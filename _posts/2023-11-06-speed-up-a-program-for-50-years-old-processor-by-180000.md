---
title: "Speed up a program for the 50 years old processor by 180000%"
date: 2023-11-06 10:24:54
slug: speed-up-a-program-for-50-years-old-processor-by-180000
layout: post
---

# Introduction

In the previous year, I wrote the program, running on Intel's first microprocessor - 4004, that [computes the first 255 digits of π](https://mark.engineer/2022/04/calculating-pi-digits-on-first-intel-microprocessor-intel-4004/) . But, unfortunately, I was not able to beat ENIAC's achievement with 2035 computed digits [^1]. So let's continue our journey. I picked the successor of 4004 - Intel's 4040. This CPU is very similar to the 4004 and extends the instruction set a bit. To make my challenge a bit more spicy, I decided to follow these rules:
1. We should compute at least 2035 digits of π faster than ENIAC did. According to the original article [^2] it took 70 hours of machine running time. So our goal is 69 hours, 59 minutes, 59 seconds or fewer.
2. We can't use extra I/O ports to expand CPU restrictions. For instance, it means that I can't have any external RAM/ROM bank switchers controlled by I/O ports.
3. It should be pure computation, with no pre-computed data tables in ROM, not even a simple multiplication table.
4. Our code should support arbitrary values for the amount of computed digits, while this value is less than or equal to a high boundary. This restriction causes avoidance of any fine-tuning for results. The solution should be generic.

*Please, take into account that this article is structured by chapters, while actual optimizations and changes have been done in another order. For example, I have optimized modular multiplication, then switched to general algorithm improvements and, because it has not been enough, I have returned to work on multiplication. However, in the article, all optimization steps for modular multiplication will be explained in the same section.*
# Technical restrictions

Even if I chose a fresher processor, it still inherits the majority of origin's limitations:
- The CPU clock rate is 740kHz, but the instruction cycle takes 8 or 16 ticks, so we have up to 92500 instructions per second.
- Very limited instruction set. We don't have multiplications, divisions, or easy memory operations. Nuff said, we don't even have shift instructions - only rotate operations.
- RAM is 1280 bytes only. Even if the datasheet contains information about lower numbers, there were real systems with 1280 bytes on board without any complex memory switch logic [^3].

But the good news is that we got some new features with 4040 [^4]:
- Now we have AND/OR instructions. You read it right, 4004 didn't have them. Unfortunatelly, these instructions are not perfect - they perform a binary operation with an accumulator and a specific register (`rr6` or `rr7` for the `AND` operation, for instance).
- The call stack grew from 3 nested subroutine calls to 7, so we can better organize our program. Still, it's a pretty low value and at some point, I exceeded it.
- Increased the amount of general-purpose registers from 16 to 24. Intel added a second bank with 8 registers. Because the ISA is the basically same and the register index is still encoded as a 4-bit number, for access to extra registers we need to switch between index register banks by executing specific instructions: `SB0` / `SB1`, which adds some inconvenience.
- ROM space has been extended as well. Now we have 2 banks of ROM with 4KiB each. Again, switching between banks is not transparent, you need to call the `DB0` / `DB1` instruction at a specific moment.

There are other changes brought by 4040, but they are not that profitable for my purpose: interrupts, single-step debugging, and separate voltage sources to reduce power consumption.
# Theory

## Algorithm choice

Let's re-evaluate what algorithms for π calculation exist.
- **Spigot algorithm** [^5]. I have already described that algorithm in my previous article. It requires keeping an array of digits in memory. The length of the array is proportional to the amount of computed digits. To have 2000+ digits, we need much more RAM.
- [**Machin-like formulae**](https://en.wikipedia.org/wiki/Machin-like_formula). Good choice, but again memory limits us. We need to perform long arithmetic and the amount of digits for operands is comparable with the desired amount of π digits. Maybe it would be possible to compute 700 or 800 digits, but there is no way we can reach our target value.
- **Improved spigot algorithm** [^6]. We don't need to keep any arrays, but we have similar considerations as for Machin-like formulas - due to computations we work with several big numbers. I was not lazy and checked how big they were - just for 255 digits of π, the algorithm requires 7KiB of RAM.
- **Bailey–Borwein–Plouffe formula** [^7]. Interesting idea, but it computes π digits in base 16 or base 2. We need base 10.
- **Plouffe's algorithm** [^8]. The original algorithm is not very practical (too slow).
- **Fabrice Bellard's algorithm is based on the same formula for π** [^9]. Simple and straightforward. Not the fastest, but much better than the version of Simon Plouffe.
- **Xavier Gourdon's algorithm** [^10]. It's even faster than Bellard's idea, but it's more complex and requires more code to be written, which is problematic with our poor ISA and limited program size (less than 8000 instructions).

From the list above, Fabrice Bellard's algorithm looks like an obvious choice, so let's research how we can write the program, based on his short paper.
## Original formula

A very interesting formula has been used as a basis for Plouffe's algorithm:
$$ \pi+3=\sum_{k=1}^{+\infty}\frac{k2^{k}}{\left(\begin{array}{c}2k\\ k\end{array}\right)} $$
You can check out a proof (and other fascinating equations) in Lehmer's article. [^11]
The round brackets expression is a [central binomial coefficient](https://planetmath.org/CentralBinomialCoefficient) and could be specified as:
$$ \left(\begin{array}{c}2n\\ n\end{array}\right)=\frac{2^n(2n-1)!!}{n!},\:n!!=n \cdot (n-2) \cdots 3 \cdot 1 $$
So the final numerical series to work with would be:
$$ \pi+3=\sum_{k=1}^{+\infty}\frac{k2^{k}}{\left(\begin{array}{c}2k\\ k\end{array}\right)}=\sum_{k=1}^{+\infty}\frac{k \cdot k!}{(2k-1)!!} $$
## Computation of the nth digit for a fraction

Let's start with the simplified problem. We have the fraction $s = \frac{b}{A}$, and we want to find its nth digit. How would we do that?
At first, we express $A$ as prime factorization:
$$ A=\prod_{i=1}^{m}a_{i},\:i \neq j \Rightarrow gcd(a_i,a_j)=1 $$
Then we can declare a few variables:
$$ b_i = b \bmod a_i;\ A_i = \frac{A}{a_i};\ A_i^{-1} = \frac{1}{A_i} \bmod a_i $$
By applying a [Chinese remainder theorem](https://en.wikipedia.org/wiki/Chinese_remainder_theorem#Existence_(direct_construction)) , we got:
$$ b' = \left( \sum_{i=1}^{m} b_i A_i A_i^{-1} \right) \bmod{A} $$
Introduce another variable $f_i = b_i \cdot A_i^{-1} \bmod a_i$:
$$ b' = \left( \sum_{i=1}^{m} f_i A_i \right) \bmod{A} $$
Because $b' = b \bmod A$, we can use it for the fractional part of $s$:
$$ \{s\} = \frac{b'}{A} = \frac{\left( \sum_{i=1}^{m} \frac{f_i}{a_i} A \right) \bmod{A}}{A} \stackrel{(1)}{=} \left\lbrace \frac{\sum_{i=1}^{m} \frac{f_i}{a_i} A}{A} \right\rbrace = \left\lbrace \sum_{i=1}^{m} \frac{f_i}{a_i} \right\rbrace $$
It should be obvious how (1) works, but for completeness, the short proof is here:
$$ \frac{x \bmod N}{N} = \frac{x - N \lfloor \frac{x}{N} \rfloor}{N} = \frac{x}{N} - \lfloor \frac{x}{N} \rfloor = \{\frac{x}{N}\} $$
Ok, now we can write the formula to find digits of $s$ from position $n$:
$$ D_n = \{s \cdot 10^n\} = \left\lbrace \sum_{i=1}^m \frac {f_i \cdot 10^n}{a_i} \right\rbrace \stackrel{(2)}{=} \left\lbrace \sum_{i=1}^m \frac {f_i \cdot 10^n \bmod {a_i}}{a_i} \right\rbrace $$
Transition (2) is here just to simplify computations: we need just the fractional part, so to avoid big numbers (and $10^n$ could be very big) we can take modulus by denominator for each fractional addend.
As you can notice, this formula is not recurrent: each term in the sum is independent and can be computed separately, so we need to store just the current intermediate result. And because of modular arithmetic, the length of variables is relatively small, hence couple of dozens of bytes is enough to keep all data.
## Computation of the nth digit for a numerical series

But the formula for π is not a regular fraction, it is represented as a numerical series, so we need to adapt math manipulations from above to another form - the partial sum of positive finite series. We have one more complication: $A_k$ (denominator of series) would be factorized to prime factors with multiplicity that could be more than 1.
Let's declare a set of prime factors $(a_1, a_2, ...,a_m)$ that have all potential factors of $A_k$ and for some $(k, i)$ multiplicity of $a_i$ in the factorization of $A_k$ could be 0 (that particular number from the set is not a factor for particular $A_k$). Also, we introduce $v_i^{max}$ - the maximal multiplicity of $a_i$ between all factorizations of $A_k$.
$$ S_N=\sum_{k=1}^N\frac{b_k}{A_k}, A_k=\prod_{i=1}^{m}a_i^{v_{i,k}}, 0 \leq v_{i,k} \leq v_i^{max},\:i \neq j \Rightarrow gcd(a_i,a_j)=1, v_{i,k} > 0 \Rightarrow a_i \nmid b_k $$
Declare variables, similar to the previous chapter:
$$ A_{i,k} = \frac {A_k}{a_i^{v_{i,k}}}\\ A_{i,k}^{-1} = \frac{1}{A_{i,k}} \bmod a_i^{v_{i,k}}\\ f_{i,k} = \begin{cases} 0 & \text{if }v_{i,k}=0\\ b_k \cdot A_{i,k}^{-1} \bmod a_i^{v_{i,k}} & \text{if }v_{i,k} > 0\\ \end{cases} $$
You can notice that we are using $b_k$ instead of $b_k \bmod a_i^{v_{i,k}}$, which are not equal even with modular multiplication. But we can do that because at the end we need just the fractional part of a number, so any integer parts we can drop early and work with numbers by modulus $a_i^{v_{i,k}}$.
By repeating logic from the previous section with the Chinese remainder theorem, we got the expression for the term of series and the whole partial sum:
$$ \frac{b_k}{A_k} = \left\lbrace \sum_{i=1}^{m} \frac{f_{i,k}}{a_i^{v_{i,k}}} \right\rbrace \\[15px] S_N = \left\lbrace \sum_{k=1}^N \sum_{i=1}^{m} \frac{f_{i,k}}{a_i^{v_{i,k}}} \right\rbrace = \left\lbrace \sum_{i=1}^{m} \sum_{k=1}^N \frac{f_{i,k}}{a_i^{v_{i,k}}} \right\rbrace $$
Next step is to use common divisor $a_i^{v_i^{max}}$, that is independent from $k$:
$$ f'_{i,k} = f_{i,k} \cdot a_i^{v_i^{max}-v_{i,k}} \\[15px] S_N = \left\{ \sum_{i=1}^{m} \sum_{k=1}^N \frac{f'_{i,k}}{a_i^{v_{i,k}} \cdot a_i^{v_i^{max}-v_{i,k}}} \right\} = \left\{ \sum_{i=1}^{m} \sum_{k=1}^N \frac{f'_{i,k}}{a_i^{v_i^{max}}} \right\} \\[15px] S_N = \left\{ \sum_{i=1}^m \frac{f_i}{a_i^{v_i^{max}}}\right\}, f_i=\left( \sum_{k=1}^N {f'_{i,k}} \right) \bmod a_i^{v_i^{max}} $$
The same trick again - because the integer part is not important, we can use numbers from residue class modulo $a_i^{v_i^{max}}$, which are not that large.
Now important moment - we can deduce that every $f'_{i,k}$ also belongs to the same residue class modulo $a_i^{v_i^{max}}$, even if it's not that obvious. We can prove that with a simplified example:
$$ (x \bmod n^p) \cdot n^{q-p} = (x - n^p \cdot \lfloor \frac{x}{n^p} \rfloor) \cdot n^{q-p} = x \cdot n^{q-p} - n^q \cdot \lfloor \frac{x}{n^p} \rfloor \\ (x \cdot n^{q-p}) \bmod n^q = x \cdot n^{q-p} - n^q \cdot \lfloor \frac{x \cdot n^{q-p}}{n^q} \rfloor = x \cdot n^{q-p} - n^q \cdot \lfloor \frac{x}{n^p} \rfloor $$
Based on that, we can write $f_i$ as (by expanding and applying reasoning above to $f'_{i,k}$ expression):
$$ f'_{i,k} = b_k \cdot A_{i,k}^{-1} \bmod a_i^{v_{i,k}} \cdot a_i^{v_i^{max}-v_{i,k}} = b_k \cdot A_{i,k}^{-1} \cdot a_i^{v_i^{max}-v_{i,k}} \bmod a_i^{v_i^{max}}\\[15px] f_i=\sum_{k=1}^Nb_k\left(\frac{A_k}{a_i^{v_{i,k}}}\right)^{-1}a_i^{v_i^{max}-v_{i,k}}\bmod a_i^{v_i^{max}} $$
And finally, we can derive a formula for digits of partial sum, starting from position $n$:
$$ D_n=\left\{ \sum_{i=1}^m\frac{f_i \cdot 10^n \bmod a_i^{v_i^{max}}}{a_i^{v_i^{max}}} \right\} $$
## Determining maximum multiplicity of a prime number in $A_k$

We can assign values to variables into the formula for digits of partial sum (you can notice that $k$ factor has been extracted from $b_k$ calculation, it's done for easier algorithm computation):
$$ b_k = k! \\ A_k = (2k-1)!!\\ v_{i,k} = v_{a_i}({A_k})\\ u_{i,k} = v_{a_i}({b_k})\\ f_i=\sum_{k=1}^N k\cdot \left( \frac{b_k}{a_i^{u_{i,k}}} \right) \cdot \left( \frac{A_k}{a_i^{v_{i,k}}} \right)^{-1} \cdot a_i^{v_i^{max}-v_{i,k}+u_{i,k}} \bmod a_i^{v_i^{max}} $$
The interesting question is how we would find the value of $v_i^{max}$ for particular $a_i$. From the problem definition from the previous section, we declare that $a_i$ does not divide $b_k$, but it's not true for our case, because $k!$ can be divisible by $a_i$ very easily. Fortunately, it affects only $v_i^{max}$ value. So to determine it, we need to find:
$$ v_{a_i}\left(\frac{A_k}{b_k} \right) = v_{a_i}\left(\frac{(2N - 1)!!}{N!}\right) $$
$v_{a_i}$ is [function of multiplicity](https://en.wikipedia.org/wiki/P-adic_valuation) . We can [expand double factorial](https://en.wikipedia.org/wiki/Double_factorial#Relation_to_the_factorial) and use [Legendre's formula](https://en.wikipedia.org/wiki/Legendre%27s_formula) to calculate it:
$$ v_{a_i}\left(\frac{(2N - 1)!!}{N!}\right) = v_{a_i}\left( \frac{(2N)!}{2^N \cdot N! \cdot N!} \right) = v_{a_i}\left( \frac{(2N)!}{N! \cdot N!} \right) = \sum_{t=1}^L \lfloor \frac{2N}{a_i^t} \rfloor - 2\sum_{t=1}^L \lfloor \frac{N}{a_i^t} \rfloor = \sum_{t=1}^L \left( \lfloor \frac{2N}{a_i^t} \rfloor - 2 \lfloor \frac{N}{a_i^t} \rfloor \right) $$
We could drop $2^N$ because it's divisible only by 2. And if we examine terms of that sum, then we can see that each particular term is 0 or 1:
$$ \lfloor \frac{2N}{a_i^t} \rfloor - 2 \lfloor \frac{N}{a_i^t} \rfloor = \lfloor \frac{N}{a_i^t} \rfloor + \lfloor \frac{N}{a_i^t} + \frac{1}{2} \rfloor - 2 \lfloor \frac{N}{a_i^t} \rfloor = \lfloor \frac{N}{a_i^t} + \frac{1}{2} \rfloor - \lfloor \frac{N}{a_i^t} \rfloor \leq 1 $$
Because we have $L = \lfloor log_{a_i}{2N} \rfloor$, then $v_i^{max} \leq \lfloor log_{a_i}{2N} \rfloor$.
## Find the upper limit for the partial sum of π series

The next question is what particular $N$ should we use for the partial sum to get the desired fractional digits. We can formalize our demand:
$$ \frac{b_M}{A_M} \cdot 10^n < 1 $$
Because we pick only fractional part, then we need to take into account terms of series, which are lower than 1 when they are multiplied by $10^n$. Let's try to solve that inequality. We would use another formula for the central binomial coefficient (it's easy to derive it from the original statement):
$$ \left(\begin{array}{c}2n\\ n\end{array}\right)=\frac{4^n(2n-1)!!}{(2n)!!} $$
And based on that, we can transform our π series term:
$$ \frac{M \cdot 2^M}{\left(\begin{array}{c}2M\\ M\end{array}\right)} \cdot 10^n = \frac{M \cdot 2^M}{\left( \frac{4^M \cdot (2M - 1)!!}{2M!!} \right)} \cdot 10^n = \frac{M \cdot 2M!!}{2^M \cdot (2M - 1)!!} \cdot 10^n $$
We can limit part of the fraction:
$$ 1 < \frac{2M!!}{(2M - 1)!!} \leq M $$
If you recall the definition of $x!!$ operator, it would be straightforward to see why that part is less than $M$ with a couple of exceptions (when $M$ is less than 4). Soon enough we would switch to logarithms, so let's use degrees:
$$ 1 < r \leq 2 \\[15px] \frac{M^{r} \cdot 10^n}{2^M} < 1 $$
Now we can calculate an estimation of $M$:
$$ \frac{M^r \cdot 10^n}{2^M} < 1 \Rightarrow M^r \cdot 10^n < 2^M \Rightarrow log_{2}{M^r} + log_{2}{10^n} < M \Rightarrow r \cdot log_{2}{M} + n \cdot log_{2}{10} < M $$
It's possible to solve it, but we don't need to do that, because anyway, we need to have some $\epsilon$ to retrieve more digits than 1 and to make sure that the fractional part has the necessary precision. So we can just add $r \cdot log_{2}{M}$ to that $\epsilon$. For our requirements ($n \leq 2048$), that addition could be a small number, and whole $\epsilon$ could be as small as 50. I picked 70 to have some spare room for imprecision in calculations with float numbers.
## Algorithm

Just implement two loops: outer loop over prime numbers and inner loop to calculate $f_i$ for particular $a_i$. There is important optimization - instead of computing factorials on each iteration, we use recurrent formulas: $A_k = (A_{k-1} \cdot (2k - 1)) \bmod m$ and $b_k = (b_{k-1} \cdot k) \bmod m$. Also, we track current information about how many times we divide $A_k$ and $b_k$ by $a_i$ to know the value of $v_{i,k}$ and $u_{i,k}$ that would be used for exponentiation of $a_i$.
$$ \begin{align*} &\begin{aligned} &N&&=\lfloor(n+\epsilon) \cdot log_{2}{10}\rfloor\\ &D_n&&=0\\ \end{aligned}\\ &for\:(a \in \mathbb{P} | 2 < a <2N)\:\{\\ &\quad\begin{aligned} &vmax &&=\lfloor \log_{a}{2N} \rfloor \\ &m&&=a^{vmax}\\ &v&&=0\\ &u&&=0\\ &f&&=0\\ &A&&=1\\ &b&&=1\\ \end{aligned}\\ &\quad for\:(k=2..N)\:\{\\ &\quad\quad\begin{aligned} &b &&= \left( \frac{k}{a^{v(a,k)}} \cdot b \right) \bmod m\\ &A &&= \left( \frac{2k-1}{a^{v(a,2k-1)}} \cdot A \right) \bmod m\\ &u &&= u + v(a, k)\\ &v &&= v + v(a, 2k-1)\\ \end{aligned}\\ &\quad\quad if\:(v - u > 0)\:\{\\ &\quad\quad\quad f = (f + k \cdot b \cdot A^{-1} \cdot a^{vmax-v+u}) \bmod m\\ &\quad\quad \}\\ &\quad \}\\ &\quad\begin{aligned} &d&&=(f \cdot 10^n) \bmod m\\ &D&&=\left\{ D + \frac{d}{m} \right\} \\ \end{aligned}\\ &\} \end{align*} $$
I am using JavaScript for a reference implementation because it's easier to translate the working HLL code to assembly, rather than writing it from scratch:
```javascript
const getNinePiDigits = (n) => {
  const N = Math.trunc((n + 20) * Math.log(10) / Math.log(2));

  let digits = 0;
  for (const a of primes.filter((prime) => prime > 2 && prime < 2 * N)) {
    const vmax = Math.trunc(Math.log(2 * N) / Math.log(a));
    const m = a ** vmax;

    let f = 0, A = 1, b = 1, v = 0, u = 0;
    for (let k = 2; k <= N; k++) {
      b = (b * (k / (a ** multiplicity(a, k)))) % m;
      A = (A * ((2 * k - 1) / (a ** multiplicity(a, 2 * k - 1)))) % m;
      u += multiplicity(a, k);
      v += multiplicity(a, 2 * k - 1);
      if (v - u > 0) {
        f = (f + (k * b * modularInverse(A, m) * (a ** (vmax - v + u)))) % m;
      }
    }

    d = (f * modularPower(10, n, m)) % m;
    digits = (digits + (d / m)) % 1;
  }

  return Math.trunc(digits * 1e9).toString().padStart(9, '0');
};
```

# Tool-stack

It has been enough to have just a compiler from assembly to bytecode for the relatively simple program from [my previous article](https://mark.engineer/2022/04/calculating-pi-digits-on-first-intel-microprocessor-intel-4004/) . But from some point, it became more and more tedious to develop code. So I had to improve my dev experience and extend the [intel 40xx tool stack](https://github.com/markablov/i40xx) with a few more tools.
## Pre-processor

I wanted to have two features:
- ability to split source code by sub-modules
- variables substitutions (a-la macros)

To support that I have added special directives and a pre-processor that searches only for those directives and outputs a single file with source code, that could be used by the compiler (and that could be pasted to the emulator UI).
Here is an example of a module, that implements some function (`foo.i4040` ):
```nasm
%define FOO_PARAM1_VAL1 0x5
%define FOO_PARAM1_VAL2 0xA
%define FOO_PARAM2_VAL1 D

foo:
  ...
  BBL 0
```

And module, that uses function `foo` :
```nasm
%include "foo.i4040"

bar:
  FIM r0, $FOO_PARAM1_VAL1 . $FOO_PARAM2_VAL1
  JMS foo
  FIM r0, $FOO_PARAM1_VAL2 . 0
  JMS foo
  BBL 0
```

Macroses were very useful for memory organization. I have re-arranged the memory layout several times and changed variable placements in memory (for example, moved a variable that keeps $N$ from memory register #5 to register #9). Without pre-processor I would have to search for numerical values and check if it's related to memory operation or just regular numeral literal.
## Compiler and linker

The first version of the compiler was very simple - sequentially picks an instruction, encodes it to bytecode, and appends it to the ROM image. When the whole bytecode is formed, the compiler updates references for jump instruction if labels have been used (here and later jump instructions include call instruction). But the Intel 40xx CPU has a few peculiarities about bytecode layout:
- Some jump instructions are short jumps. It means that they can only jump to locations, that are inside the same ROM page (0x100 bytes) as the instruction itself. For example, it's possible to use such instructions for jumps from `0x2AA` to `0x255`, but not possible to `0x655`.
- Short jumps have another specialty - they should not be placed at the last bytes of the ROM page. If the jump instruction is located at address `0x7FF` or `0x7FE` and tries to perform a short jump to `0x7BB`, the actual jump would be done to `0x8BB` because of how the CPU works.

With the initial compiler, I solved those problems either by manual arrangement of routines or by padding problematic functions with `NOP` instructions to fix misalignments. It worked while the code size was not that large, but the code bloated, and doing all those refinements became a nightmare.
Other considerations for better multi-stage compilation were:
- Intel 4040 started to support two ROMs, and for big programs, we want to use both banks.
- Need to be able to put particular routines to specific locations.

Going to spend some time and dig into details about that two features, required from the compiler.
As I have mentioned before, it's not that simple to jump from one ROM bank to another. You should do something like that:
```nasm
__location(0x0:0x00)
entrypoint:
  JMS output
  DB1
  JMS foo
  HLT

output:
  LDM 0xF
  WMP
  BBL 0

__rom_bank(0x1)
foo:
  JMS output
  DB0
  NOP
  BBL 0
```

We can see here:
- `entrypoint` is placed at bank #0, at location `0x000`, it's the actual first instruction, that would be executed by the CPU after RESET
- `foo` is placed at bank #1 at a non-specified location, the linker would choose the best place for that block.
- The `output` routine is shared between banks. Linker would have to duplicate that routine in both banks.
- `NOP` instruction before `BBL`. `DBx` instruction switches bank on the 3rd cycle, so we need to spend an extra cycle before the actual flow control instruction.

We want to be able to specify location not only for entry point but for switch tables. The Intel 40xx ISA has instruction `JIN rX` , which jumps to address, contained by a register pair. It allows me to execute different code blocks, based on register value without necessary to have many comparisons. I have used that trick in several places, here is an example of optimized 4bit by 4bit division, when we have special routines to divide operands by 1, 2, 3, ...:
```nasm
__location(00:0x10)
div4x4_1:
  XCH rr12
  LDM 0x0
  XCH rr13
  BBL 0

# INPUT:
#   acc - dividend
#   rr10 - divisor
#   rr11 - 0x0
# OUTPUT:
#   rr12 - quotient
#   rr13 - reminder
__location(00:0x18)
div4x4:
  JIN r5

__location(00:0x20)
div4x4_2:
  RAR
  XCH rr12
  TCC
  XCH rr13
  BBL 0

...
```

The compilation pipeline from source code to ROM image has 4 stages:
1. Preprocessing. Have described that stage in the previous paragraph.
2. Parses a source code using simple grammar and produces a list of encoded instructions with metadata (like the type of instruction or source code line number) and a list of references: if the instruction has a reference to the location of another instruction (jumps by label, for example), then we need to know that to update the reference in the final image.
3. Unites instructions into code blocks. These blocks are not always whole functions, but they could be smaller pieces of code. Even if instructions are semantically linked (implement the same function), they could be spread into the ROM and don't have to be placed nearby. So the block is the smallest set of instructions that could be executed sequentially without a guaranteed jump (program counter change). ![](../../../images/2023/11/code_blocks.png)
4. Now we have a list of blocks and references between blocks. Based on that, we form superblocks - a set of blocks which has short jumps to each other. They should be considered together, because we have extra limitations for their placements, while blocks with long references are untangled and could be placed in any relative locations from each other. Unfortunately, we can't apply an algorithm that solves the classical [bin packing problem](https://en.wikipedia.org/wiki/Bin_packing_problem), because superblocks can cross ROM page boundaries, despite that they have short jumps:

![](../../../images/2023/11/rom_layout.png)
That's why I chose a simple and naive way - we have a cursor, that points to the ROM location and goes forward to the end of ROM. On each iteration, I try to insert some superblock at that cursor (and increase the cursor by superblock size), and if we can't do it for any superblock, then move the cursor by 1 byte forward. Few notes:
- Blocks with fixed locations could be placed instantly, without any extra analysis.
- Try to fit the biggest superblock first, usually, it's harder to fit large routines.
- There could be gaps between fixed blocks, so even if the ROM cursor is moving forward only, we can try to fill gaps with small blocks, before performing the main arrangement loop.

The order of blocks inside the superblock also could vary and can affect the possibility of placement. So we need to check all permutations of blocks inside the superblock.
The amount of permutations is $n!$ and the superblock could have 10+ blocks, hence checking all variants is a heavy operation that could take minutes. That's why I have added a compilation cache - we know the best position and permutation for superblock. If it has not been changed, then we can reuse that stored information.
## Emulator and profiler

I need a way to debug the program and its parts. It's much easier to do with proper tools. I already had a simple Intel 4004 emulator, but I had to extend it a bit:
- Important to be able to use the emulator outside the browser. I wrote a bunch of tests to validate that even basic functions work correctly.
- Added support of Intel 4040 features: new instructions, extended registers bank, multiple ROM banks.
- Some tests require pre-defined RAM snapshots, so we need to be able to pass some initial RAM state, instead of clean RAM banks.
- Another important feature was profiling. I wanted to understand, what functions consume a majority of the time. With emulation, it's possible to have instruction-based profiles, but for me it was good enough to have function scope.
- GUI started to support source maps. We want to be able to determine the source code line, based on the ROM address of currently executed instruction. Because the ROM layout is not that flat anymore, the linker should produce a source map.
- Also I have added watchers to the GUI - you can declare variables for debug purposes, and watch how they are changing live. For instance, we can have a 2-digit number, with a lower digit in rr5, and a high digit placed in status character #0 from memory register #1. Even if it's possible to find out the value of that number with registers and RAM panels, it could be not very easy. So you can specify watcher `[71]:s0,rr5` and see the value of that number while debugging the program.

[![](../../../images/2023/11/emulator-1024x615.png)](../../../images/2023/11/emulator.png)
# Basic assembly implementation

To be honest, I underestimated the computation complexity of the task. So my first implementation was relatively dumb and had almost no optimizations. But there are still a few points to mention.
## Preparation

#### Number limits

At first, I determined the bit depth for numbers. The highest $n$ is 2043, then $N$ is less than 7000, and modulus $m$ is no more than 14000. It means that for the majority of operations 14 bits (2^14 gives a high boundary as 16384 for numbers) are enough. But of course, I want to work with CPU words, so our target bit-width is 16 bits for almost all variables. That's good because it fits to status characters of memory registers.
Let me remind you of the structure of RAM in Intel 40xx systems. We have up to 8 RAM banks. Each RAM bank consists of 16 memory registers. Each memory register is split into two parts: 16 main 4-bit characters and 4 status 4-bit characters. An important difference is how memory access is organized for different kinds of characters. In most cases, access to main characters is slower:
```nasm
FIM r0, 0xF0   # loads 0xF0 to r0
SRC r0         # selects register F and main character 0
RDM            # loads the selected main character to the accumulator
XCH rr2        # rr2 = accumulator
FIM r0, 0xF1   # loads 0xF1 to r0
SRC r0         # selects register F and main character 1
RDM            # loads the selected main character to the accumulator
ADD rr2        # accumulator = main character 0 + main character 1
```

Status character registers could be accessed by a special set of instructions without necessary to select a new character every time:
```nasm
FIM r0, 0xF0  # loads 0xF0 to r0
SRC r0        # selects register F and main character 0
RD0           # loads status character 0 to the accumulator
XCH rr2       # rr2 = accumulator
RD1           # loads status character 1 to the accumulator
ADD rr2       # accumulator = status character 0 + status character 1
```

That's why storing variables inside status characters is better than storing them in main characters.
OK, now I know bit-depth and can make other estimations. Prime numbers could be computed once, using [sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes) . The rough estimate for the amount of prime digits below $2N$ is 2000. How are we going to store them? We can use bitmap: if the bit on position $x$ in RAM is set to 1, it means that $x$ is prime. We have 10240 bits in RAM, while we need 14000 (high boundary for $2N$). But we don't care about even numbers, they are never prime, except 2, which is out of our interest. Hence, we need just 7000 bits.
#### Logarithms and float numbers

What are the other tricky parts? Logarithms... I need to know $log_{2}{10}$ for $N$ computation. Even if I can calculate it just once, taking a logarithm is still a very hard operation for the CPU without float numbers and native division. Yes, I can just use a pre-calculated exact constant, but it would be against the rules. Instead of that, I use approximation: $\log_2{10}\approx\frac{93}{28}$. Precision should be enough. We are not afraid of overshooting $N$, it would just add more work, but would not affect correctness.
Another logarithm to evaluate is $log_{a}{2N}$. But, fortunately, we need just the integer part, so we can have a simple loop $v^{max} = \max {v: a^v < 2N}$.
As I have mentioned, we don't have float numbers, but $D$ is the fractional part of the float number. I solved it by keeping that number in simple fixed decimal point form - just multiply every $d$ by $10^{15}$ to have 15 fractional digits. 9 digits would be our result and 6 digits to preserve precision to avoid cumulative rounding errors.
#### Modular arithmetic

Time to think about modular arithmetic. The simplest way would be just to perform an operation (multiplication/exponentiation) and then divide it by modulus to get a reminder. But I decided to do it a bit smarter - by using the binary method. There is a multiplication property of modular arithmetic:
$$ (a \cdot b) \bmod m=(a \bmod m \cdot b \bmod m) \bmod m $$
Knowing that, we can write such recursive equation for $f(x, y) = (a \cdot b) \bmod m$:
$$ f(x, y) = \begin{cases} 0 & \text{if }b = 0\\ f(2 \cdot a \bmod m, b / 2) & \text{if } b \text{ is even}\\ (a + f(a, b - 1)) \bmod m & \text{if } b \text{ is odd}\\ \end{cases} $$
A similar idea we can use for [modular exponentiation](https://en.wikipedia.org/wiki/Modular_arithmetic#Example_implementations) :
$$ a^b \bmod m = \begin{cases} 1 & \text{if $b = 0$}\\ (a^\frac{b}{2})^2 \bmod m & \text{if } b \text{ is even}\\ ((a^\frac{b-1}{2})^2 \cdot a) \bmod m & \text{if } b \text{ is odd}\\ \end{cases} $$
The last operation is modular inversion. Because we can be sure that $A$ and $m$ are co-prime numbers, we can use [Euler's theorem](https://en.wikipedia.org/wiki/Euler%27s_theorem) :
$$ A^{\phi(m)} \equiv 1 \pmod m\\ A^{\phi(m)-1} \equiv A^{-1} \pmod m $$
From definition $m = a^{vmax}$, so we can calculate Euler's totient function easily:
$$ \phi(m) = \phi(a^{vmax}) = a^{vmax} - a^{vmax - 1} $$
## Results

How close that code would be to target 70 hours? Very, very far. I was not able to finish the emulation, but I used polynomial fit after computing ~800 digits to get some estimation. And it was oppressive **14.5 years** . Looks like I started a long journey to improve that time.
# Optimize, optimize, optimize!

## Algorithm improvements

### "Concurrent" computations

Before digging into low-level optimizations, I wanted to examine the algorithm more scrupulously. I had no huge expectations because Fabrice Bellard has a brilliant mind and chances that he missed some obvious point were low. But he solved another problem than me - to determine the *nth* digit of π, while I was looking for *n* digits of π. Maybe we can reuse some computation results before runs? In-memory memoization cache could not be large: just around 300 entries if we are using 16bit numbers as cache keys and cache values. We need to do something smarter. The key observation is that $N$ has no high limit for particular $n$. It just adds more precision, but $D$ still would be correct. So we can compute common $f_i$ for all digit positions below $n$, with $N$ computed for highest required $n$. And only specific operation for different digit positions would be $D =\left\lbrace D + \frac{(f_i \cdot 10^{digitPosition}) \bmod m}{m} \right\rbrace$.
Because we still would compute $f_i$ for high $N$ at some point, then extra work would be just that unnecessary $D$ updates for low digit positions when we don't need such precision. We keep intermediate values for $D$ as fixed-precision numbers with up to 15 digits after the decimal point and they need more space to be kept - whole 16 words (8 bytes). Hence, in the best case, we can keep 160 such numbers while we need 2052 / 9 = 228 entries. It's not a big deal, because we can split computation into two intervals (0, 1026) and (1026, 2052). Of course, it would be better to run that loop just once, but memory limitations hit us again.
Small change is required for HLL implementation:
```javascript
    for (let i = 0; i < digits.length; i++) {
      d = (f * modularPower(10, startingPosition + i * 9, m)) % m;
      digits[i] = (digits[i] + (d / m)) % 1;
    }
```

After applying that idea to my assembly code I have been pleased with **85x** improvement - **64 days** instead of many years. But still need to make it 20x times faster!
### Memory layout for prime numbers

We started to store 100+ 8-byte numbers in memory and now we are running out of memory for our prime numbers bitmap. Let's revisit it then.
Instead of a regular sieve, we can use [segmented sieve](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes#Segmented_sieve) . The idea is to have an initial segment of prime numbers lower than $\sqrt{2N}$ (let's call it segment size) and be able to generate the following segments based on that initial data. Segment size equals 118, so we know two facts about the initial prime segment: 1) prime numbers fits 1 byte (2 words), and 2) there is less than 30 prime number lower than 118. Based on that, we need just 30 bytes for the initial segment.
The following prime segments can be organized in a very compact manner as well. The difference between the first and last prime numbers in the following segment is no more than segment size by definition. Hence, we can keep the lowest value of the segment as is (16bit number) and have a bitmap for 59 entries (again we don't need even numbers to be represented in our map). So even if we spent the whole word for bitmap value it would take only 32 bytes in total.
### Fixed-precision numbers format

We don't keep $D$ in BCD format (for example, in that format `1234` would be represented as `0x1234` in memory instead of `0x4D2` ) because of memory limitations. So it makes less sense to have a decimal point instead of a hexadecimal point. Hence, instead of multiplying $D$ by $10^{15}$, we can multiply it by $2^{50}$, which is just bit shift operation.
### Fast path for big primes

I was still skeptical about the possibility of reaching such performance gain only with code optimizations, so I continued to investigate the basis of the algorithm. And quickly discovered that for primes bigger than $\sqrt{2N}$ we have $v^{max} = 1$. It allowed me to cut a lot of edges. $v$ for that case has a binary nature - it is either 0 or 1, but more exciting is that $v$ alternates between two states with continuous sequences. $v$ switches to 1 when $a_i | 2k-1$ and then switches back to 0 when $a_i | k$. Here is a simple visualization of alternation.
![](../../../images/2023/11/v_switch.png)
The basic HLL code would look like that:
```javascript
  do {
    // iterate until next k, that satisfies (2 * k - 1) % a === 0
    for (; (2 * k - 1) % a !== 0; k++) {
      b = (b * k) % a;
      A = (A * (2 * k - 1)) % a;
    }

    // v becomes 1 here
    b = (b * k) % a;
    A = (A * ((2 * k - 1) / a)) % a;
    f = (f + modularInverse(A, a) * b * k) % a;
    k++;

    // iterate until next k, that satisfies k % a === 0
    for (; (k % a) !== 0; k++) {
      b = (b * k) % a;
      A = (A * (2 * k - 1)) % a;
      f = (f + modularInverse(A, a) * b * k) % a;
    }

    // now v becomes 0
    b = (b * (k / a)) % a;
    A = (A * (2 * k - 1)) % a;
    k++;
  } while (k <= N);
```

We already excluded exponentiation of $a$ and many unnecessary comparisons, but we can do better than that:
- We can get rid of division. The outer loop step is $a$, so `k / a` and `(2 * k - 1) / a` increments by 1 and 2 correspondingly for each outer loop iteration.
- We don't need exact values of $k$ for computations, we can use $k \bmod a$ to have faster modular multiplication with lower numbers.
- At the beginning of outer loop we know that $k \bmod a = 1$, hence we have constant sequence of multiplications for $b$ with $v = 0$. It could be calculated outside loop: $b_{v=0} = (\frac{a - 1}{2} + 1)! \bmod a$. Starting from $k \bmod a = \frac{a-1}{2} + 1$ we are switching $v$ to be 1.

The improved (by performance, not by readability) version is:
```javascript
  let multiplierForZeroV = factorial(((a - 1) / 2) + 1) % a;
  let k = ((a - 1) / 2) + 1;
  let reducedCoefA = 3;

  do {
    // iterate until next k, that satisfies (2 * k - 1) % a === 0
    for (reducedCoefB = 2; reducedCoefB <= (a - 1) / 2; reducedCoefB++) {
      A = (A * reducedCoefA) % a, reducedCoefA += 2;
    }
    b = (b * multiplierForZeroV) % a;

    // v becomes 1 here
    A = (A * multiplierA) % a, multiplierA += 2;
    f = (f + modularInverse(A, a) * b * reducedCoefB) % a;
    k++, reducedCoefB++;

    // iterate until next k, that satisfies k % a === 0
    reducedCoefA = 2;
    for (; reducedCoefB < a; k++, reducedCoefB++) {
      A = (A * reducedCoefA) % a, reducedCoefA += 2;
      b = (b * reducedCoefB) % a;
      f = (f + modularInverse(A, a) * b * reducedCoefB) % a;
    }
    // have a check of the loop boundary now to avoid redundant first loop
    k += (a + 1) / 2;
    // now v becomes 0
    b = (b * multiplierB) % a, multiplierB++;
    A = (A * reducedCoefA) % a, reducedCoefA += 2;
    // try to keep the factor lower than "a"
    if (reducedCoefA > a) {
      // factor is incremented by 2, so it could be bigger than "a" just by 1
      reducedCoefA = 1;
    } else {
      A = (A * reducedCoefA) % a;
      reducedCoefA += 2;
    }
  } while (k <= N);
```

It was a good observation, but it gave me just dozens of percents as a performance gain, while I need thousands. Time to look into basic operations.
### Profiling

Usually, when you have performance problems you want to determine bottle-necks and hot code. So I started with gathering information about typical execution profiles for heavy computation routines. It was easy enough to produce stack traces, compatible with [flamegraph](https://github.com/brendangregg/FlameGraph) tool.
[![](../../../images/2023/11/flamegraph_noopt-1024x158.png)](../../../images/2023/11/flamegraph_noopt.png)
Now I have a starting point. After each optimization, I re-evaluated the profile to determine target functions, that are worth checking.
## Modular inversion

From the first flamegraph, it was very clear that I needed to do something with modular inversion. Looks like a "fast" implementation based on Euler's theorem is not that fast. After I changed the algorithm to regular division-based [extended Euclidean](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm#Modular_integers) , I got very promising numbers. But then I thought about [binary version](https://en.wikipedia.org/wiki/Binary_GCD_algorithm) of the same algorithm. High-level code is simple:
```javascript
const modularInverse = (a, m) => {
  if (a === 1) {
    return 1;
  }

  let rx = a, ry = m;
  let v = 1, u = 0;

  while (rx % 2 === 0) {
    rx = rx >> 1;
    v = (v % 2 === 0) ? (v >> 1) : (v + m) >> 1;
  }

  while (ry > 1) {
    if (ry % 2 === 0) {
      ry = ry >> 1;
    } else {
      if (ry < rx) {
        [rx, ry] = [ry, rx];
        [v, u] = [u, v];
      }
      ry = (ry - rx) >> 1;
      u = (u - v) % m;
    }

    u = (u % 2 === 0) ? (u >> 1) : ((u + m) >> 1);
  }

  return u < 0 ? (u + m) : u;
}
```

Translation to assembly was also straightforward. Maybe just one interesting point was about the halving negative numbers (all other functions operate only with positives). Surprisingly, it was more efficient in i40xx assembly than I thought:
```nasm
  LD rr3       # load highest word of number
  RAL          # put sign bit to carry to be able to shift negative numbers
  LD rr3       # again load the highest word of the number
  RAR          # shift it right, but because carry stores sign a bit
               # it would be propagated back to the highest bit of word
  XCH rr3      # save updated word
  ...
```

I have not continued to develop a faster version even if it was possible by using [various tricks](https://www.researchgate.net/publication/304417579_Modular_Inverse_Algorithms_Without_Multiplications_for_Cryptographic_Applications) . Some of them was easy enough to be implemented even with i40xx ISA.
## Modular exponentiation

We are using that function in two places: to compute $10^x$ when updating $D$ and to compute $a^{v^{max} - v + u}$. Because we know that $x$ is increasing by 9 at every iteration, we can avoid exponentiation with just 3 modular multiplication by 1000 (because our multiplication routine is working with 16bit numbers we can't multiply by $10^9$):
```javascript
    let powered10 = modularPower(10, startingPosition, m);
    for (let i = 0; i < digits.length; i++) {
      const d = (powered10 * f) % m;
      digits[i] = (digits[i] + (d / m)) % 1;
      powered10 = (powered10 * 1000 * 1000 * 1000) % m;
    }
```

Exponentiation also is used in cases when $v^{max} > 1$. Even if now we know that there are not so many such cases we can improve it as well. We can limit $v^{max} < \lfloor log_{a_0}{2N} \rfloor$, hence we have a power value no more than 7. Moreover, $a^{v^{max} - v + u} < a^{v^{max}}$, so we don't need *modular* exponentiation at all. And because we have just a maximum 7 values of $a^t$, we can store them into a small cache table.
## Division

After I had added special flow for $v^{max} = 1$, amount of divisions decreased drastically, but I did division optimization before that change, so I left it in place.
### 4bit x 4bit division

The foundation of multi-word division is a 4bit x 4bit routine. Even naive code with a subtraction loop was not that slow:
```nasm
div4x4:
  FIM r6, 0x00
div4x4_loop:
  SUB rr11
  JCN nc, div4x4_return
  CLC
  ISZ rr12, div4x4_loop
div4x4_return:
  ADD rr11
  XCH rr13
  CLC
  BBL 0
```

17 cycles on average for all combinations of dividend and divisor.
In the chapter, related to compiler and linker, I have shown an approach with 15 specific code blocks for 15 divisors (divide by 0?!). Some blocks were not able to fit in 0x10 bytes (the gap between two addresses in our switch table), but because of conditional jumps inside them, it was possible to arrange all cases on the same ROM page. That optimization cut 6 cycles from each function call. Not much, but it's a 35% performance gain for that small routine.
### 4bit x 4bit multiplication

Another building block for division is 4bit x 4bit -> 8bit multiplication. A naive implementation of 4x4 division was fast, but we can't say the same for basic 4x4 multiplication. The addition of the first term in the loop, counted by the second term was not very efficient (~70 cycles on average). Then it is worth spending memory on the multiplication table! We are going to use a similar trick with the switch table to have 16 code blocks for 16 multipliers. Most of them would fetch data from memory, but we can implement cases for 0/1/2/8 multipliers by regular code to save some RAM.
```nasm
__location(01:0x00)
mul4x4_0:
  FIM r6, 0x00                                         # low, high = 0
  BBL $BANK_WITH_VARIABLES

# INPUT:
#   acc - first factor (a)
#   rr10 - second factor (b)
#   rr11 - 0x0
# OUTPUT:
#   rr12 - low word of product
#   rr13 - high word of product
__location(01:0x08)
mul4x4:
  JIN r5

__location(01:0x10)
mul4x4_1:
  XCH rr12                                              # low  = a
  LDM 0x0
  XCH rr13                                              # high = 0
  BBL $BANK_WITH_VARIABLES

__location(01:0x20)
mul4x4_2:
  RAL
  XCH rr12
  TCC
  XCH rr13
  BBL $BANK_WITH_VARIABLES

__location(01:0x30)
mul4x4_3:
  XCH rr12
  LDM $BANK_WITH_MUL_TABLE_FOR_FACTOR_3
  DCL
  SRC r6
  RD2
  XCH rr12                                              # low
  RD3
  XCH rr13                                              # high
  BBL $BANK_WITH_VARIABLES
```

### 16bit x 16bit division

For multi-word division I have used well-known Knuth's Algorithm [^12]. But I can make it more specific and have different code paths for combinations of dividend and divisor sizes. For instance, there is a case when both divisor and dividend are 3 words long or the dividend is 2 words long, but the divisor is just 1 word. We have just 10 combinations (divisor is always lower than dividend) and only 3 of them would require to follow Algorithm D: 16x8, 16x12, and 12x8 divisions. In all other cases, we can cut edges and perform more efficient calculations, without speculating over estimated quotient digits.
## Modular multiplication

After all the improvements from previous chapters, modular multiplication became the main computation abyss - it occupied more than 60% of execution time. And I spent a lot of time polishing that operation. I have tried several approaches and some optimizations have been replaced by more efficient alternatives. I would describe the final form of that function.
### Algorithm

There is a temptation to use the [Montgomery form](https://en.wikipedia.org/wiki/Montgomery_modular_multiplication) to perform modular multiplications, especially because we can keep that form for our inner loop over $k$ and perform conversion back just before computing chunks of digits. But due to Montgomery modular multiplication, we are doing 3 regular multiplications, and even with optimizations you would have to perform $\frac{13n^2}{8} + n$ single-word multiplications[^13], where $n$ is the amount of words in number ($n = 4$ in our case). So it means 30 4x4 multiplications, and each costs 12 cycles, which gives us an estimation for 360 cycles only for multiplications, and with 150 additions (formula from the same paper) it under-performs the current solution.
What about straightforward fast multiplication with modular reduction? It still requires at least 16 calls to 4x4 multiplication routines [^14] only for the multiplication step. And the reduction is much more costly. Not a solution. This paper also uses the Karatsuba algorithm for multiplication, but only for big numbers - for small values, this algorithm is worse than just regular scanning multiplication.
So we are going to use just regular binary multiplication, but with a bunch of low-level hacks:
```javascript
const modularMultiply = (a, b, m) => {
  let multipliedFactor = a;
  let result = 0;

  for (let shiftedFactor = b; shiftedFactor > 0; shiftedFactor = shiftedFactor >> 1) {
    if (shiftedFactor & 0x1 === 0x1) {
      result = (result + multipliedFactor) % m;
    }

    multipliedFactor = (multipliedFactor + multipliedFactor) % m;
  }

  return result;
};
```

### Lazy modular reduction

The idea is that we don't need to have a `result` to belong to the least residue system modulo $m$ before returning the final value. It means that we don't need to do modular reduction on each step. But we still need to perform that operation to prevent integer overflow (we operate with 16-bit numbers). Initially, modular reduction after each addition could be done with single subtraction: `a = a > m ? a - m : a` , because each addend was less than $m$. With a lazy approach, it's not true anymore and we have to do proper modular reduction. But we can do it a bit smarter - overflow could potentially happen with 4-word numbers only, so we can create a lookup table with pre-computed multipliers for $m$, that could be subtracted from the reduced number to make it smaller and still keep it positive and congruent. Of course, that reduction would not be perfect, but it's good enough to prevent overflows. LUT is very simple:
$$ x = \overline{x_3 x_2 x_1 x_0}, x_3 \rightarrow \lfloor \frac{x_3 \cdot 16^{3}}{m} \rfloor \cdot m $$
That correction would give us a rough value. But for a small modulus, it could be still a big divergence. We don't want to create more LUTs (memory is not infinite), but we can keep one more number:
$$ \lfloor \frac{F00_{16}}{m} \rfloor \cdot m $$
If the reduced number, even after subtracting the value from LUT is still more than `0x1000` , we can reduce it by that constant. It allows us to always have 3-word numbers after rough reduction.
By that moment in the article, I have described all caches and tables that would be kept in RAM, so I can show you the final memory layout:
![](../../../images/2023/11/memory_3.png)
Coloring legend:
- Green section is values for $D_n$. 114 registers.
- Purple section - that LUT for modulus multipliers.
- Red section - multiplication tables. Remind you, that we need just 2 words for table entry, so each RAM bank can have a table for 2 multipliers. 6 banks = 12 multipliers, we excluded tables 0, 1, 2, and 8 because we compute the result of multiplication by that factors in the code.
- Orange section - table with pre-computed exponents for current prime number. Can keep up to 8 4-digit entries.
- Yellow section - primeness map for the current segment, we use it to iterate through prime numbers.
- Blue section - the initial segment of prime numbers. Can have up to 32 2-digit numbers.
- White regions - free to use for variables.

Pretty dense layout, but still better than in my previous work.
### Extend laziness for the whole computation routine

Then I thought about the moment when we need to have the exact value from the least residue system modulo $m$. And looks like we need it only on the very last step - before calculating $\frac{d}{m}$. All other computations could be done with congruent numbers because they are performed via modular arithmetic. It reminds me of Montgomery form a bit - when you convert input values to that form, do different operations, and just before returning the final result you need to convert it back.
Not having exact values would affect performance in some way - values would be bigger than they could, so more unnecessary logic to execute. But overall benefit from avoiding proper modular reduction is much greater.
### Addition instead of subtraction

Maybe it would sound strange, but addition requires fewer instructions than subtraction, because of the inverted carry flag (borrow for subtraction). When you subtract one number from another, the borrow flag would be set if there is *no borrow* . So if you are doing subtraction of multi-word numbers you need to inverse borrow flag by using extra instruction `CMC` to let the subsequent call of `SUB` use the right information about borrow from the previous digit:
```nasm
# compute [rr0, rr1] = [rr0, rr1] - [rr2, rr3]
LD rr0
SUB rr2    # acc = rr0 - rr2, borrow flag is 0, if there was borrow
CMC        # inverse borrow flag
XCH rr0
LD rr1
SUB rr3    # acc = rr1 - rr3 - borrow, borrow flag is 0, if there was borrow
CMC        # inverse borrow flag
XCH rr3
```

Addition does not require to correct carry flag:
```nasm
# compute [rr0, rr1] = [rr0, rr1] + [rr2, rr3]
LD rr0
ADD rr2    # acc = rr0 + rr2, carry flag is 1, if there was carry
XCH rr0
LD rr1
ADD rr3    # acc = rr1 + rr3 + carry, carry flag is 1, if there was carry
XCH rr3
```

We can replace the subtraction of $x$ with the addition of $16^3 - x$ ([two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) for $x$) for 4-word numbers and the result would be the same but with less amount of instructions. It's a generic observation, but it's most profitable for modular multiplication because of how many times this operation is performed.
### Switch tables again

We have no more than 16 loop iterations, so we can unroll the loop body and check each bit of the 2nd factor:
```nasm
  # assume that rr6 has b[0]
  LDM 0x1
  AN6
  JCN z, mulMod_skipMultiplierBit0       # if ((b >> 0) & 0x1)
  JMS mulMod_updateResult                #   res = res + multipliedFactor
mulMod_skipMultiplierBit0:
  JMS mulMod_updateMultipliedFactor      # multipliedFactor = multipliedFactor * 2

  LDM 0x2
  AN6
  JCN z, mulMod_skipMultiplierBit1       # if ((b >> 1) & 0x1)
  JMS mulMod_updateResult                #   res = res + multipliedFactor
mulMod_skipMultiplierBit1:
  JMS mulMod_updateMultipliedFactor      # multipliedFactor = multipliedFactor * 2

  # ... 14 more checks for particular bits
```

But we can go further and for specific 4-bit words we know the sequence of operations. For instance here are `res` and `multpliedFactor` updates, when the current digit of factor is 5:
```nasm
__location(0x2:0x50)
mulMod_processNibble_5:                   # 0b0101
  JMS mulMod_updateResult
  JMS mulMod_updateMultipliedFactor
  JMS mulMod_updateMultipliedFactor
  JMS mulMod_updateResult
  JMS mulMod_updateMultipliedFactor
  JUN mulMod_updateMultipliedFactor
```

We can extend that idea and have another switch table for the last digit because we don't need to sustain `multpliedFactor` after we process the highest non-zero bit:
```nasm
__location(0x3:0x50)
mulMod_processLastNibble_5:                   # 0b0101
  JMS mulMod_updateResult
  JMS mulMod_updateMultipliedFactor
  JMS mulMod_updateMultipliedFactor
  JUN mulMod_updateResultLast
```

### Perform 3 or 4 consequential multiplied factor updates as a single operation

The last optimization was trivial - we can compute `16 * multipliedFactor` and `8 * multipliedFactor` by using word or bit shifts. But need to check that we would not overflow 16bit. If there is such risk, then we need to do a fallback and call `mulMod_updateMultipliedFactor` several times.
### Merge subroutines

Often after the temporal result update, we need to update the multiplied factor and in the code we have such snippets:
```nasm
  JMS mulMod_updateResult
  JMS mulMod_updateMultipliedFactor
```

The idea is to have a single routine `mulMod_updateResultAndFactor` that has combined code from `mulMod_updateResult` and `mulMod_updateMultipliedFactor` . But why it's better? Math is simple - we had 2 subroutine calls and 2 returns from them, with that change we would have 1 subroutine call and 1 return. It allows us to save 3 cycles for each such case. It could sound like a very marginal number, but please take into account that `mulMod` takes 60% of the whole execution time and it's a relatively small function, so every non-executed instruction wins us several minutes.
### Faster check about potential overflow of multiplied factor

As I have mentioned before, we reduce the multiplied factor only if it could overflow our 16-bit variable. To check potential overflow, a very simple reasoning is used: because the multiplied factor is doubled, we can check that the highest nibble is always less than `0x8`
```nasm
  XCH rr2                                           # multipliedFactor = multipliedFactor + multipliedFactor
  AN7                                               # check if previous value of multipliedFactor[3] < 4, then new multipliedFactor[3] < 8
  JCN z, mulMod_updateResultAndFactor_return        # if (multipliedFactor[3] < 0x8)
```

This is a small and efficient check - just 3 cycles (`JCN` instruction takes 2 machine cycles to execute). But every cycle matters, that's why another trick I have used - replace the `AN7/JCN` pair with single instruction `JIN` . We occupy another ROM page with a switch table. `JIN r0` jumps to address, encoded in `r0` , and if `rr0` represents the most significant digit of multiplied factor, then we can split code execution into two paths: when `rr0 < 0x8` and `rr0 >= 0x8` . Of course, it's very inefficient in terms of ROM space, but we still had some room.
```nasm
  JIN r1

__location(0x4:0x00)
mulMod_updateMultipliedFactor_factor0:
  BBL 0

__location(0x4:0x10)
mulMod_updateMultipliedFactor_factor1:
  BBL 0

...

__location(0x4:0x80)
mulMod_updateMultipliedFactor_factor8:
  SRC r1
  RD0
  ADD rr0
  XCH rr0
  RD1
  ADD rr1
  XCH rr1
  RD2
  ADD rr4
  XCH rr4
  RD3
  ADD rr2
  XCH rr2
  CLC
  BBL 0

...
```

### Optimizations that have been removed from the final code

There were few working ideas, but because of different reasons that are not present in the code (mostly because other optimizations were better and because of ROM space):
- Special code path for 12-bit modulus. Because at some moment I have stopped to keep variables less than $m$, this separation by modulus size became useless.
- Rough comparison of numbers with modulus. Before lazy reduction, we wanted to check if a number is bigger than $m$ to do subtraction. This comparison was costly (because it was subtraction itself and if the operand was lower than $m$, this operation was just a waste of time). To perform a rough comparison we could use another lookup table:

```nasm
SRC rX      # first register in register pair contains the address of LUT, second register - multipliedFactor[3]
RDM         # read isLessThanModulus = LUT[multipliedFactor[3]]
JCN z, ret  # return if (isLessThanModulus)
```

But with lazy reduction, we don't care anymore if the number would be higher than the modulus.
- Another idea was to avoid unnecessary subtractions was to keep temporal result as negative. We can easily detect a sign of the number, and if it becomes positive, then just subtract the modulus. Before returning, always add $m$.
- The amount of performed operations depends on the most significant one bit of 2nd factor. So we want to use the lowest factor as the shift factor. It requires swapping operands if 2nd factor is bigger than 1st. It is good optimization, but the results were marginal - it was hard to choose multiplications that could use these features.
- We could have a bit finer reduction even for the lazy version - add another lookup table for numbers less than `0x1000`. But it requires more RAM and helped a lot only with a small modulus.

## Results on emulator

After all that jiggery-pockery, the whole program finished by 69h 29m 02s and I even got the right digits for π. Great success. Here is one of the latest flamegraphs.
[![](../../../images/2023/11/flamegraph_final-1024x158.png)](../../../images/2023/11/flamegraph_final.png)
Now it's time to run it on real hardware.
# Hardware

I used almost the [same schematic](https://github.com/markablov/i4040-sbc/tree/main/schematic) as for my previous project. The main differences are:
- Solder the stm32 chip in place with all necessary auxiliary components, instead of using a stm32 board, plugged in a DIP socket into the main board.
- Single USB-based power source instead of external 15V.
- Replace the UART interface with USB.

Other than that design is almost identical: LM339 and CD40109 are used as level shifters.
[![](../../../images/2023/11/schematic-1024x710.png)](../../../images/2023/11/schematic.png)
# Firmware

Again, the [core of firmware](https://github.com/markablov/i4040-sbc/tree/main/firmware) has been inherited from the original i4004-sbc board, but I had to make a few corrections.
Intel 4040 memory interface is a bit more complex and has an extra pin to select the ROM bank. That's why we have to do more stuff in between cycles. Additionally, the i4004-sbc board was configured to run on the lowest specified clock boundary of the Intel 4004 CPU - 625kHz, while I need the highest possible frequency (740kHz) to fit into my tight target timing.
At some moment I found out that I couldn't use the clocking timer (stm32 timer that produces clocks for the CPU) interrupt, because of latencies. I switched to a simple pin polling inside the main loop:
```c
  while ((OUT_4040_PHI1_GPIO_Port->IDR & OUT_4040_PHI1_Pin) == OUT_4040_PHI1_Pin);

  OUT_TEST1_GPIO_Port->ODR ^= OUT_TEST1_Pin;
  handleCyclePhi1Falling();
  OUT_TEST1_GPIO_Port->ODR ^= OUT_TEST1_Pin;
```

Pin toggling is here to track how much time the stm32 spends inside a subcycle.
Another consideration was the USB interface instead of UART. Interrupts had to be disabled due i4004 program run, because of very accurate timings - each subcycle is 1350ns long, and stm32 had to fit into that window. With 168Mhz frequency for the stm32 core, we have just 200 spare stm32 cycles to read pins, perform necessary calculations, and set output pins.
So all interaction with the PC is frozen after the `START` command with ROM dump is received. I/O instructions from intel 4040 are tracked and logged into the in-memory buffer. When the CPU reaches a halt state (new `HLT` instruction), we send the accumulated buffer with output to the controlling program on the PC.
# Results

After fixing all issues with hardware and firmware, I was able to run i4040 programs. At first, I found out that I made a heart-breaking mistake in my calculations due to tests on the emulator - I used the wrong coefficient to compute the estimated running time, based on cycle count: `95000 cycles/sec` instead of `92500 cycles/sec` . It meant that the real running time on the real i4040 was just a bit more than the target 70 hours, so I had to return to the source code and find even trickier optimizations.
Out of interest, I started with the smaller task: computation of the first 255 digits of π.
```
[2023-10-31T12:08:05.527Z] START command has been received, RAM dump size is 8192 bytes
[2023-10-31T13:13:08.988Z] Program has been finished
[2023-10-31T13:13:08.994Z] Instruction cycles processed = 00000000158651E4
[2023-10-31T13:13:08.995Z] Output from i4040 (263 nibbles):
[2023-10-31T13:13:08.995Z] 3 1 4 1 5 9 2 6 5 3 5 8 9 7 9 3 2 3 8 4 6 2 6 4 3 3 8 3 2 7 9 5 0 2 8 8 4 1 9 7 1 6 9 3 9 9 3 7 5 1 0 5 8 2 0 9 7 4 9 4 4 5 9 2 3 0 7 8 1 6 4 0 6 2 8 6 2 0 8 9 9 8 6 2 8 0 3 4 8 2 5 3 4 2 1 1 7 0 6 7
[2023-10-31T13:13:08.995Z] 9 8 2 1 4 8 0 8 6 5 1 3 2 8 2 3 0 6 6 4 7 0 9 3 8 4 4 6 0 9 5 5 0 5 8 2 2 3 1 7 2 5 3 5 9 4 0 8 1 2 8 4 8 1 1 1 7 4 5 0 2 8 4 1 0 2 7 0 1 9 3 8 5 2 1 1 0 5 5 5 9 6 4 4 6 2 2 9 4 8 9 5 4 9 3 0 3 8 1 9
[2023-10-31T13:13:08.995Z] 6 4 4 2 8 8 1 0 9 7 5 6 6 5 9 3 3 4 4 6 1 2 8 4 7 5 6 4 8 2 3 3 7 8 6 7 8 3 1 6 5 2 7 1 2 0 1 9 0 9 1 4 5 6 4 8 5 6 6 9 2 3 F
```

As you can see, it took 01hr 05m 03s in comparison with 03hr 31m 13s from the run of the spigot algorithm on i4004. That's not a fair comparison of any kind, because CPU frequencies have been different, and more importantly is that I spent no time on optimization in the previous project. But still was kinda curious to see that numbers.
And finally, I started computation of 2048+ digits with ETA as a bit less than 70hrs. I have prepared an Intel 4040 CPU with a couple of tiny radiators and it successfully finished the job:
[![](../../../images/2023/11/final_output-1024x228.png)](../../../images/2023/11/final_output.png)
The final time is 69h 28m 31s and my journey completes here.
PS: The attentive reader could notice that the time from the hardware run is different from the emulator, even if the amount of executed cycles is close. It's because the clock signal, generated from stm32 timers is not 100% accurate, and the output frequency is about 740.1kHz, which is higher than the specified 740kHz. So I would say that the Intel 4040 even got overclocked.
# References

[^1]: [Brian J. Shelburne, "The ENIAC's 1949 Determination of π"](https://ieeexplore.ieee.org/document/5999630) [↩](#fnref-120-1)
[^2]: [George W. Reitwiesner, An ENIAC Determination of π and e to more than 2000 Decimal Places](https://www.jstor.org/stable/2002695) [↩](#fnref-120-2)
[^3]: [Intel Intellec 4/Mod 40 Microcomputer Development System Reference Manual](http://datasheets.chipdb.org/upload/wepwawet/INTELLEC%204.pdf) [↩](#fnref-120-3)
[^4]: [Intel, "MCS-40 User's Manual For Logic Designers"](http://www.bitsavers.org/components/intel/MCS40/MCS-40_Users_Manual_Nov74.pdf) [↩](#fnref-120-11)
[^5]: [Stanley Rabinowitz and Stan Wagon, "A Spigot Algorithm for the Digits of π"](https://www.maa.org/sites/default/files/pdf/pubs/amm_supplements/Monthly_Reference_12.pdf) [↩](#fnref-120-4)
[^6]: [Jeremy Gibbons, "Unbounded Spigot Algorithms for the Digits of Pi "](https://www.cs.ox.ac.uk/people/jeremy.gibbons/publications/spigot.pdf) [↩](#fnref-120-5)
[^7]: [David Bailey, Peter Borwein and Simon Plouffe, "On the Rapid Computation of Various Polylogarithmic Constants"](https://www.davidhbailey.com/dhbpapers/digits.pdf) [↩](#fnref-120-6)
[^8]: [Simon Plouffe, "On the computation of the ݊n'th decimal digit of various transcendental numbers"](https://arxiv.org/ftp/arxiv/papers/0912/0912.0303.pdf) [↩](#fnref-120-7)
[^9]: [Fabrice Bellard, "Computation of the n'th digit of pi in any base in O(n^2)"](https://bellard.org/pi/pi_n2/pi_n2.html) [↩](#fnref-120-8)
[^10]: [Xavier Gourdon, "Computation of the n-th decimal digit of π with low memory"](http://numbers.computation.free.fr/Constants/Algorithms/nthdecimaldigit.pdf) [↩](#fnref-120-9)
[^11]: [D. H. Lehmer, "Interesting Series Involving the Central Binomial Coefficient"](https://www.jstor.org/stable/2322496) [↩](#fnref-120-10)
[^12]: [Donald E. Knuth, "Art of Computer Programming, Volume 2: Seminumerical Algorithms", 4.3. Multiple Precision Arithmetic](https://www.amazon.com/Art-Computer-Programming-Seminumerical-Algorithms/dp/0201896842) [↩](#fnref-120-12)
[^13]: [Hwajeong Seo, Zhe Liu, Yasuyuki Nogami, Jongseok Choi, Howon Kim, "Hybrid Montgomery Reduction"](https://dl.acm.org/doi/pdf/10.1145/2890502) [↩](#fnref-120-13)
[^14]: [Michael Hutter, Peter Schwabe, "Multiprecision multiplication on AVR revisited"](http://mhutter.org/papers/Hutter2014MultiprecisionMultiplicationOn.pdf) [↩](#fnref-120-14)
