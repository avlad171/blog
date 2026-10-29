Welcome, traveller! If it is reverse engineering experience that you seek, you have come to the right place! This is a
page where I will share stories about my journey through the field so it may come in useful for anyone wishing to attempt
similar adventures. No prior experience required.


# Foreword
I will try to stick to the essentials as much as I can, avoiding the typical boilerplate often seen in academia and such.
This is not institutional research or anything formal for that matter, and I believe that just hitting arbitrary word count
statistics using textbook definitions of a "control-flow-graph" or a "linear-sweep" disassembler is just making the useful
information harder to read and process.

I encourage everyone to enjoy the content in any way they like, from a Sunday's read to replicating what is presented
here byte-by-byte, taking breaks, detours and individual experiments to understand or expand upon a specific concept.

There will be links to literature or other blogposts and you are always welcome to ask questions. For example, 
[this book](https://en.wikipedia.org/wiki/Reversing:_Secrets_of_Reverse_Engineering) is an excellent overview of software reverse engineering.

# Abstraction
To quote [wikipedia](https://en.wikipedia.org/wiki/Abstraction_(computer_science)),
an abstraction provides access while hiding details that otherwise might make access more challenging. This or a similar
definition is usually encountered in software engineering, especially object oriented programming. In order to make a
product easy to use and maintain it is important to be able to see the bigger picture, "abstracting" away the inner workings.

The job of a fridge is to keep its insides cooler than the room it is in. It provides this specific cooling functionality
which allows us to store food for longer. A cook working in a restaurant knows there is a space cooled to 5 C where he can store
X kg of food. How much electricity it uses, or the chemical formula of the coolant are usually not thought of.

The business owner can sometimes be interested in its power consumption, but it is usually only the technician that knows
the circuit diagram or other inner workings of the machine, because this information is useful to him and his job. Everyone
can be interested of the internals for various reasons, like thinking about the environmental impact of the coolant gas,
or just because they are passionate about that subject, but all of this is not required for the job.

Software is different in the fact that you can't touch it. We call such concepts abstract. It is a product you (sometimes)
pay money for and, in exchange, it performs a task. To do its job it has to run on a computer, usually on top of an OS. This machine
has various types of memory, a data bus, a CPU, expansion cards, interrupts, peripherals. The OS has drivers that interact with
the hardware, libraries to interact with the drivers, APIs, privilege levels, memory management, programming language runtimes
and so on. This is already a lenghty list of jargon. Not everyone using a computer can, or is willing to, understand each component.
Even if they were it would be very impractical.

# Reverse engineering a base-3 virtual machine

Because why not.

![A dank meme](three/fly-bitch-v0-awk8n1citzfz.webp)

I could spend hours talking about various aspects of reversing, but there is more juicy stuff out there. As every video
game teaches, a harder boss gives more XP so we'll attack the hardest.

THE PROBLEM IN QUESTION

Its name is "[three](https://2019.ctf.link/internal/challenge/d31f5063-3e0c-4295-9707-2143035d3f0a/)" from hxp 36c3 CTF
held in 2019. It has 1000 points, the largest amount. It is rated the highest difficulty and has only been solved by 1
team during the contest. As far as I know, I am the second entity to attack it.

![Picture with description](three/challenge.png)

# Initial assessment

I start with static analysis. I don't know what I'm up against and usually there's a lot of information that can be
extracted from the metadata alone. This is a 64-bit elf which loads fine in IDA, disassembles and decompiles without
too much trouble, and is quite small (44KB).

Loading is important because it rules out most file format trickery or any techniques which would turn this into a "misc"
challenge. The size indicates there's no runtimes or interpreters embedded, and also that it is not a statically linked
stripped binary which requires searching the correct libc version, usually by digging through the author's Twitter to 
find a screenshot of his/her Linux setup.

The import table is quite small, only containing a few standard libc functions, like `puts`, `fgetc` and `fputc`. A limited
number of imported functions can sometimes mean executable packing, or other obfuscation which takes upon itself to resolve
the external symbols; you can read more about imports [here](https://sandsprite.com/CodeStuff/Understanding_imports.html).

There are only two useful strings in this program: `":("` and `":)"`. Both are referenced in the main function.

The number of functions is again limited. Not much is happening outside main() and the control flow is easily recovered by IDA.
Or is it? *spoiler alert*

![Main CFG](three/cfg.png)
This is anything but nice to look at. It is very discouraging, and we didn't even get to see the assembly yet.
I can not stress enough how important a first impression is.

There's a couple of things that could be happening here:
1. A complex algorithm, possibly well known, maybe a nonstandard or esoteric implementation. Straightforward reversing, but a lot of instructions, time consuming.
2. Some form of IL level obfuscation, like OLLVM. I say IL and not assembly because the code decompiles fine.
3. Control flow flattening, may include a dispatcher or duplicated nodes.
4. A virtual machine

or more. In real life it could be some of the above, at the same time or layered.
So where do we start? There's a couple options, depending on what the goal of the reverse engineering is.

As per <insert paper> ... In this program we don't need to look very far. Second block reads input already, and we easily
recognize a 4-clause if: ![Prologue](three/prologue.png). It checks for hxp{.....} which is the flag format! It is probably
safe to assume that the goal here is to extract a secret. A crackme. The next block is ugly, we have SSE instructions.

4 blocks down and there is an even uglier one. Very long and ends into a switch statement with some cases also represented
by huge monoliths.

Here some people may be tempted to start reversing instruction by instruction. Go look "online" what the SIMD instructions do
and try to make sense. This approach will most likely not work in a CTF (72 hours or less), as we've barely seen 15% of the CFG.
So it has to be something smarter.

One obfuscation idea is to replace constants or simple code with complex computations that have constant results. In that
way the reverse engineer spends a lot of time analysing noise instead of the actual program. Thankfully it is the 21st
century and things that can pattern match DAGs exist for free.
![LLVM meme 1](three/llvm_meme1.jpg)

# Lifting to LLVM
Although lifting was not a new concept, 2019 marks the release of [SATURN -- Software Deobfuscation Framework Based on LLVM](https://arxiv.org/abs/1909.01752).
Two well-known security researchers formalize and publish a method of attacking compiler based obfuscation. There were
attempts to use compiler frameworks before against protectors who de-optimize code, moving data around or unfolding constants,
but these suffered from a fundamental flaw: lack of a proper model of the host architecture.

Let me explain. You have a block of assembly instructions, and you'd like to only collapse an expression. How do you model
this? Do you nop the basic block and replace with a function call? What about jumps? Loads/stores? If I store a value at 
0x5788 and read it 3 blocks below do I count it as dead store or live? Can I still fold the computation if a partial result
is written somewhere in memory?

SATURN uses Remill to lift the whole compile unit, in order to recover its CFG. It slices the program counter at the end of each block.
Only then could all the optimizations be applied. <rephrase>

More modern approaches slice at the instruction level instead. This is not the purpose of this post, but I'm mentioning it
here so you can check it out if interested.

With all this hype around it, [mcsema](https://github.com/lifting-bits/mcsema) seemed like a logical next step.

As you might have already guessed it didn't work.

Not only it was extremely hard to compile and setup, but it produced a program more complex than the original. LLVM just
isn't built for reversing. I don't want to discredit it; in fact there will be multiple future blog posts using it to lift.
But the reality is often disappointing. Machine code instructs the CPU to do very precise, specific tasks, while high level
languages use abstractions to make the code easy to understand by humans. LLVM does neither.

[Other people](https://www.babush.me/hitcon-ctf-2020-writeup.html) have similar experiences, especially with CTF binaries.
In fact, not long after this CTF, a [secret.club](https://secret.club) member created his own intermediate language to
attack VMProtect. That should tell you something about LLVM.

You either need to model each instruction with its side effects for 100%[citation needed] correctness, or make assumptions
and risk ending up with semantically different code. Knowing what to focus on and what to discard is the real problem which
no one seems to talk about. More on this in the future. Also, just so I don't forget: program equivalence is undecidable!

# Retdec

B-but [retdec](https://github.com/avast/retdec) is a real thing! It is a decompiler powered by LLVM and it works!
That is true, but you know what also works? HexRays decompiler from within IDA.

During the CTF I didn't have enough time to dig through remill, mcsema and retdec to see what are the similarities and
differences between them. Obviously this problem can be solved by lifting to an open source IR/framework.

# HexRays decompiler

One of the first tools I ran on this program was IDA's plugin decompiler. It ran cleanly with only one undefined variable.
The main difference between this and LLVM for this program consists of the xmm instructions. Hexrays keeps them as intrinsics,
while remill does not.

<insert code example>

After looking through this code I noticed some SSE instructions repeating, or being used more than others. It is still 4000
lines of pseudo-C, and I have no control over their formatting.

I felt already that the xmm intrinsics are the key to this challenge. The next few hours during the CTF were spent googling
SSE obfuscators like ssexy (don't plug that into Google Images btw) and function un-inlining. This led me down a graph theory
rabbit hole along with some shady IDA plugins not compatible with the then-current version.

Trying to match the SSE patterns by hand was tedious and error-prone. My second approach was with regular expressions with
semi-manual formatting, as one can't parse C with regex.

<insert example + explaination>

# An unsolved challenge

This is where I gave up during the CTF. At this point I could only see 3 viable solutions:
1. A cheese of some sort, a shortcut to only get the flag without reversing too much
2. Lifting to LLVM the proper way.
3. A modified C compiler frontend to write transformations over C code.

The CTF ended and the author posted the intended solution for this problem. A VM computing a block cipher (well known algorithm),
flattened main function and matching the SSE intrinsics. These were used to defeat Angr, so, in theory, a fork of angr
with support for those instructions, or similar software, could solve this problem.

4. An updated framework, such as angr or triton.

I will leave here the capture of mIRC conversation. 

/spoiler
36C3-IRCBOT12/30/2019
<jjk> d3xf: backdoored binary, backdoor overwrites md5_transform to have only 12 rounds
<h4x0r|3167> what was three challenge?
<h4x0r|3167> custom vm?
<hlt_> h4x0r|3167: yeah, custom vm
<h4x0r|3167> Then method for solving is only analyze binary fully and write disassembler?
<h4x0r|3167> It was very hard to understand what xmm operation does
<hlt_> basically the VM is compiled with attribute((flatten)) so you want to find repeating patterns in the binary and reverse those
<hlt_> For XMM-anything I always recommend https://software.intel.com/sites/landingpage/IntrinsicsGuide/#!
<hlt_> I may post source at some point, but here is the rough idea: http://homepage.divms.uiowa.edu/~jones/ternary/bct.shtml
<hlt_> (fun fact: Dragon Sector had a similar architecture at their CTF finals, independently of this challenge)
<h4x0r|3167> Thanks for your recommendation, Then each vm opcode does simple operation? like xor, add, mul
<hlt_> yeah pretty much
<h4x0r|3167> cause it has so many xor, andn, or, ...
<hlt_> to avoid angr (I didn't realize it can't load XMM stuff anyways) the code that runs in the VM reads the flag in 8-byte blocks, encrypts each, and compares it to a stored result
<h4x0r|3167> Then encryption algorithm was known-algorithm?
<h4x0r|3167> or custom?
<hlt_> SAFER K-64
<hlt_> I needed something with single-byte operations
<hlt_> because the VM registers are only 6 trits (-243 to 243, or 729 values)
<hlt_> so Wikipedia list of block ciphers to the rescue ;)
<h4x0r|3167> thanks a lot, anyway that challenge was so nice and hard
<h4x0r|3167> have you any plan for publishing source code or write-up?
/spoiler end

All this is just on paper, there's no official solve, and the fact that nobody tried it (to this day
there's no writeups on ctftime) makes me believe it's still exceptionally hard. The problem? Lack of tools.

IDA and Hexrays have a horrible undocumented API, LLVM is hard to even compile, angr does not support SSE2 and to write passes
over C you need to basically write a compiler.

# Cheesing
I want to reverse this thing because the goal of this post is reverse engineering, but cheesing is a valid tactic in CTFs
so it is worth mentioning. Also, learning how tasks are designed and what to expect from them will make you a better player.
Call this meta-reverse-engineering if you like.

I googled SAFER-K64 implementations, s-boxes and other constants, to no avail. The VM uses balanced ternary, so I was careful
to modify [findcrypt](https://github.com/polymorf/findcrypt-yara) to look for such a format. It does not work out of the box,
but it might actually be possible! See annex.

# Reversing
What we know so far:
- [Spoiler] A base 3 system, probably implemented using a VM
- SSE2 instructions which are arcane and not supported by mainstream tools
- Lifting to LLVM doesn't do anything out of the box, some kind of ad-hoc passes are required
- Flag is 48 bytes long

The author confirmed the (balanced) ternary VM, using SSE2 to defeat angr and obviously the flag was released
once the CTF ended. Most people at this point lose interest, because the challenge was "spoiled" or something.
It is still very hard and very interesting to look at, for my goal aren't CTF points but knowledge. "How do you 
reverse such a thing by yourself?" So the spoilers aren't very useful, and won't be mentioned again in this post.

I already had a hunch about a base 3 virtual machine. If it wasn't this, what else could it have been, then?
Some kind of geometry problem? Polynomials? A weird crypto algorithm? Base-3 crypto algorithms do exist, by the way.
In fact, a reversing challenge featuring exactly this was given at DefCon Quals 2026. A blog post on that one is also coming ;)

There's so many questions and zero answers, but I have to start somewhere. Let's look again at the only (interesting) place
fgets() reads input: ![input code](three/processing_input.png)

Notice how these two switch cases are duplicated. Code is exaclty the same: transforming binary input
into balanced ternary. It looks like a function has been inlined here, or the code could have even been
copy-pasted in the source. It appears in too few places to be a fully unrolled program, so we can deduce
some sort of flattening is taking place. A virtual machine has a handler for every operation it supports.
In this example, there is fgetc() + binary_to_BCT and some other operation + binary_to_BCT, so just the conversion
is not a handler by itself.

VM or not one thing becomes clear: I need to attack this problem at the block / expression level!

# Constants

These are the important constants used by the VM machinery. I will just list them for now, and will come back to them once
they are fully analyzed, understood and renamed. Here are, in their full glory, 18 XMM scary monsters:

.rodata:0000000000009130 xmmword_9130    xmmword 10000000000000000000000000000h
.rodata:0000000000009140 xmmword_9140    xmmword 20000000000000000000000000000h
.rodata:0000000000009150 xmmword_9150    xmmword 10001000100000000000100000000h
.rodata:0000000000009160 xmmword_9160    xmmword 40004000400040004000400040004h
.rodata:0000000000009170 xmmword_9170    xmmword 30003000300030003000300030003h
.rodata:0000000000009180 xmmword_9180    xmmword 10001000100010001000100010001h
.rodata:0000000000009190 xmmword_9190    xmmword 10004001000400100040000000000h
.rodata:00000000000091A0 xmmword_91A0    xmmword 0FFFEFFFEFFFEFFFEFFFEFFFEFFFEFFFEh
.rodata:00000000000091B0 xmmword_91B0    xmmword 100030009001B005100F300000000h
.rodata:00000000000091C0 xmmword_91C0    xmmword 0FFFFFFFFFFFFFFFFFFFFFFFF00000000h
.rodata:00000000000091D0 xmmword_91D0    xmmword 0FFFF0000000000000000000000000000h
.rodata:00000000000091E0 xmmword_91E0    xmmword 20002000200020002000200020002h
.rodata:00000000000091F0 xmmword_91F0    xmmword 0FFFFFFFFFFFFFFFFFFFFFFFFFFFFh
.rodata:0000000000009200 xmmword_9200    xmmword 20003000300030003000300030003h
.rodata:0000000000009210 xmmword_9210    xmmword 0FFFFFFFFFFFFFFFF0000000000000000h
.rodata:0000000000009220 xmmword_9220    xmmword 0FFFFFFFF0000FFFFFFFFh
.rodata:0000000000009230 xmmword_9230    xmmword 20002000200030003000200030003h
.rodata:0000000000009240 xmmword_9240    xmmword 0FFFFFFFFFFFFFFFFFFFF000000000000h

The most important questions you should ask here are the size of the integers and their endianess. An xmmword is 128 bits wide.
It can represent a single 128bit integer, a vector of 16 u8s, 8 u16s, 4 u32s or 2 u64s. Some of these already seem to encode a strange pattern: they look like vectors of u16.

IDA does not easily expose such views in the disassembler. In the debugger you can select the data type, but disassembly just treats them like integers.
Why is this important? Because it drops the leading zeros! I wish I knew this earlier, before seriously attacking this problem. Here are the same constants represented as u16 vectors.

xmmword_9130    xmmword 0001 0000 0000 0000 0000 0000 0000 0000h
xmmword_9140    xmmword 0002 0000 0000 0000 0000 0000 0000 0000h
xmmword_9150    xmmword 0001 0001 0001 0000 0000 0001 0000 0000h
xmmword_9160    xmmword 0004 0004 0004 0004 0004 0004 0004 0004h
xmmword_9170    xmmword 0003 0003 0003 0003 0003 0003 0003 0003h
xmmword_9180    xmmword 0001 0001 0001 0001 0001 0001 0001 0001h
xmmword_9190    xmmword 0001 0004 0010 0040 0100 0400 0000 0000h
xmmword_91A0    xmmword FFFE FFFE FFFE FFFE FFFE FFFE FFFE FFFEh
xmmword_91B0    xmmword 0001 0003 0009 001B 0051 00F3 0000 0000h
xmmword_91C0    xmmword FFFF FFFF FFFF FFFF FFFF FFFF 0000 0000h
xmmword_91D0    xmmword FFFF 0000 0000 0000 0000 0000 0000 0000h
xmmword_91E0    xmmword 0002 0002 0002 0002 0002 0002 0002 0002h
xmmword_91F0    xmmword 0000 FFFF FFFF FFFF FFFF FFFF FFFF FFFFh
xmmword_9200    xmmword 0002 0003 0003 0003 0003 0003 0003 0003h
xmmword_9210    xmmword FFFF FFFF FFFF FFFF 0000 0000 0000 0000h
xmmword_9220    xmmword 0000 0000 0000 FFFF FFFF 0000 FFFF FFFFh
xmmword_9230    xmmword 0002 0002 0002 0003 0003 0002 0003 0003h
xmmword_9240    xmmword FFFF FFFF FFFF FFFF FFFF 0000 0000 0000h


I removed the address because it is encoded in the default name IDA gives them. Now you can see that a lot of these vectors have their last two elements 0.


# Expressions

##  1. Extract lane
The first expression that screams at you is
```C
epi16 = _mm_extract_epi16(_mm_and_si128(_mm_slli_si128(_mm_load_si128((const __m128i *)&xmmword_9150), 10), (__m128i)xmmword_91D0), 7);
```
The last constant is always 7, but the "10" can be replaced with 8, 6, 4, 2 or 0. Therefore, I'll note it with "n".

# TODO add pic

HexRays splits it on 3 lines which makes it more difficult to read. Regardless, I rebuild this one by hand to understand it and then it can be automated later.
Instruction reference is here: `https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html`.

`__m128i _mm_load_si128 (__m128i const* addr)` is just a load, it can be dropped.
`__m128i _mm_slli_si128 (__m128i a, int imm8)` Shifts "`a` left by `imm8` bytes while shifting in zeros" and returns the result.
The microcode only specifies that the last 8 bits of imm8 are used, and that its value is truncated at 16. What I am interested in is the last expression: `dst[127:0] := a[127:0] << (tmp*8)`

`__m128i _mm_and_si128 (__m128i a, __m128i b)` is the regular bitwise and
`int _mm_extract_epi16 (__m128i a, int imm8)` is the tricky one here. For one, it does not treat data as single integer (si128) but as packed 16bit integers. See the `si` and `pi` suffix? You'll get to know them by the end of this writeup!
It does 
```verilog
dst[15:0] := (a[127:0] >> (imm8[2:0] * 16))[15:0]
dst[31:16] := 0
```

This last instruction returns an `int`, even though only the lower 16bit half carries any data. All the rest return vectors. Let's now combine the two expressions in a C-like syntax:
```C
int ex1 = (vector << (8 * n) & 0xFFFF 0000 0000 0000 0000 0000 0000 0000) >> (7 * 16);  // n = 10, 8, 6, 4, 2, 0`
```
It's easy to see now that `xmmword_91D0` is a bitmask and this whole operation extracts the "n/2"th element of the vector.

[Spoiler] With the power of retrospective I can tell you that these vectors encode trits. This first expression extracts a single trit from its unpacked xmmword form.

[Note] It was at this point that I created the first translation rule over the C code. I'm continuing with the patterns as it is the focus of this section. When reversing, I did not know if the idea actually works and I wanted to see if it actually appears multiple times along with some simplification before investing effort into it. Moving on.

## 2. Test zero
Immediately after the 6 consecutive instances of expression 1 you can find the simpler
```C
v650 = _mm_xor_si128(_mm_cmpeq_epi16(v641, (__m128i)0LL), (__m128i)-1LL);
```

The only tricky thing here is the packed compare `__m128i _mm_cmpeq_epi16 (__m128i a, __m128i b)`. Instead of producing a single true / false, it produces 8 of them.
From Intel's manual, `dst[i+15:i] := ( a[i+15:i] == b[i+15:i] ) ? 0xFFFF : 0`. The second operand is the constant 0, so it can be rewritten as `dst[i+15:i] := ( a[i+15:i] == 0 ) ? 0xFFFF : 0`. Now, this gets xor'ed with 128bit `-1`, which is just `0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF`.
0 xor 1 is 1 and 1 xor 1 is 0, so this is just a negation. Folding the operations results in `dst[i+15:i] := ( a[i+15:i] == 0 ) ? 0 : 0xFFFF`. This does not produce a boolean True or False, and often this expression is fed into `_mm_movemask_epi8(test_0(x))` to ask "any lane still set?".

## 3. Unpack
This is the hardest pattern in the entire binary. Get ready because there's a lot to unpack (pun intended).

```C
v17 = (__int64)v640;
*v640 = *(_QWORD *)v13;
v18 = *((_DWORD *)v13 + 2);
v20 = 16 * (unsigned __int8)v18;

_mm_unpacklo_epi64(
    _mm_unpacklo_epi32(
        v14, //always equal to 0
        _mm_insert_epi16(
            _mm_cvtsi32_si128(v20 >> 10),
            BYTE1(v20) & 3,
            1)),
    _mm_unpacklo_epi32(
        _mm_insert_epi16(
            _mm_cvtsi32_si128(((unsigned __int16)v20 >> 6) & 3),
            ((unsigned __int16)v20 >> 4) & 3,
            1),
        _mm_insert_epi16(
            _mm_cvtsi32_si128(((unsigned __int16)(v20 | ((int)(unsigned __int8)v17 >> 4)) >> 2) & 3),
            ((unsigned __int16)v20 | ((int)(unsigned __int8)v17 >> 4)) & 3,
            1)));
```

First, `_mm_cvtsi32_si128` appears 3 times. Combined with `_mm_unpacklo_epi32` and `_mm_unpacklo_epi64` it hints at a recursive structure.
`__m128i _mm_cvtsi32_si128 (int a)` is nothing special, just zero extending a 32bit integer to 128 bits. The key to this problem lies in its inputs. It might seem random, but this is the first appearance of this pattern in the binary, and it is a weird one due to how trits are stored in this program. There are better, cleaner example down in the decompilation,
so let's abstract the inputs for now:

```C
_mm_unpacklo_epi64(
    _mm_unpacklo_epi32(
        0, 
        _mm_insert_epi16(
            _mm_cvtsi32_si128(a),
            b & 3,
            1)),
    _mm_unpacklo_epi32(
        _mm_insert_epi16(
            _mm_cvtsi32_si128(c & 3),
            d & 3,
            1),
        _mm_insert_epi16(
            _mm_cvtsi32_si128(e & 3),
            f & 3,
            1)));
```

This is much more readable. The next logical question would be "What is the input type?"  `_mm_cvtsi32_si128` takes in 32bit ints. What about `_mm_insert_epi16`?

```
__m128i _mm_insert_epi16 (__m128i a, int i, int imm8)
dst[127:0] := a[127:0]
sel := imm8[2:0]*16
dst[sel+15:sel] := i[15:0]
```
It takes in ints too, but only cares about the lower 16 bits. This pattern hardcodes the selection index to 1, so this just updates the second lane in a vector.
`_mm_cvtsi32_si128` creates a vector and sets lane 0 to `e`. Then _mm_insert_epi16 takes this vector and sets lane 1 to `f`. Ignore the &3 for now, this just tells you data is 2 bits wide, the data source is the only thing that matters for now.

At this point the vectors are (top to bottom):
`0 0 0 0 0 0 0 0`,
`0 0 0 0 0 0 b a`.
`0 0 0 0 0 0 d c`.
`0 0 0 0 0 0 f e`.
Next, the program just interleaves them:
```
__m128i _mm_unpacklo_epi32 (__m128i a, __m128i b)
DEFINE INTERLEAVE_DWORDS(src1[127:0], src2[127:0]) {
	dst[31:0] := src1[31:0] 
	dst[63:32] := src2[31:0] 
	dst[95:64] := src1[63:32] 
	dst[127:96] := src2[63:32] 
	RETURN dst[127:0]	
}
```
Starting from LSB, the order is src1 -> src2 -> src1 -> src2. In case of these vectors it now looks like this (32bit lanes instead of 16bit):
`00 00 ba 00`
`00 00 fe dc`

`__m128i _mm_unpacklo_epi32 (__m128i a, __m128i b)` is almost the same. It takes the lower half of src1 and the lower half of src2 to produce `fedc ba00`.
Represented as u16 vector this reads `f e d c b a 0 0`. It does not look like unpacking, but exactly the opposite.

Remember the inputs. They were `v20` and `(unsigned __int16)v20 | ((int)(unsigned __int8)v17 >> 4)`. v20 = 16 * (unsigned __int8)v18, an u8 multiplied by 16. Whatever value this operation takes is stored 2/3 in v20 and 1/3 in v17.
At this point it is just a theory, and not a convincing one. v18 and v17 come from different places. It is possible one of them is just 0.

I chose to go with this theory because there are much cleaner examples down the line:
```C
 _mm_unpacklo_epi64(
   _mm_unpacklo_epi32(
     v14,   // constant 0
     _mm_insert_epi16(_mm_cvtsi32_si128((v42 >> 10) & 3), HIBYTE(v42) & 3, 1)),
   _mm_unpacklo_epi32(
     _mm_insert_epi16(_mm_cvtsi32_si128((v42 >> 6) & 3), (v42 >> 4) & 3, 1),
     _mm_insert_epi16(_mm_cvtsi32_si128((v42 >> 2) & 3), v42 & 3, 1))),
```
Now you might ask, why did I choose such a cryptic pattern as the foremost example? It was the first one in the program, and also that it requires to see past the noise, which is the point of reverse engineering. The meaning of a pattern can be adjusted during the reversing process. In fact, that's exactly what's next.

## 4. Pack OR Compute Scalar

The next pattern that repeats semi-frequently is this one. In the decompilation, it first appears with ct_9190. I only observed the 2 variants when my matcher did correctly not match it,
```C
v37 = _mm_madd_epi16(vector, (__m128i)ct_9190); // OR ct_91B0); [SPOILER] if ct_9190 it creates a packed trit, if ct_91B0 it computes a scalar (i.e. BCT to binary)
v38 = _mm_extract_epi16(
        _mm_packs_epi32(
          v14,  // constant 0
          _mm_add_epi32(v37, _mm_add_epi32(_mm_slli_si128(v37, 8), _mm_slli_si128(v37, 4)))),
        7);
```
I'll write these constants down again:
```
ct_9190 = 0x 0001 0004 0010 0040 0100 0400 0000 0000
ct_91B0 = 0x 0001 0003 0009 001B 0051 00F3 0000 0000
```

At first glance, `__m128i _mm_madd_epi16 (__m128i a, __m128i b)` looks like a packed (lane by lane) 16bit addition. It actually is multiply-and-add and the operation size is a bit unclear:
```verilog
dst[i+31:i] := SignExtend32(a[i+31:i+16]*b[i+31:i+16]) + SignExtend32(a[i+15:i]*b[i+15:i])
```

Let's see an example. Note `vector`'s lanes t1 to t8 (MSB -> LSB). The operation looks like:
```declarative
ct_9190 = 0x 0001  ,  0004  ,  0010  ,  0040  ,  0100  ,  0400   ,  0000  ,  0000
vector  =    t1    ,  t2    ,  t3    ,  t4    ,  t5    ,  t6     ,  t7    ,  t8
v37     =    t1*1  +  t2*4  ,  t3*16 +  t4*64 ,  t5*256+  t6*1024,  0     ,  0
```

`_mm_add_epi32` is the plain lane-by-lane 32bit addition. Remembering that `_mm_slli_si128` is just a left shift with distance measured in bytes, the addition subtree becomes `v37 + (v37 << (8 * 8) + v37 << (4 * 8))`.
```declarative
v37       =    t1*1   + t2*4   ,   t3*16  + t4*64  ,   t5*256 + t6*1024,   0               ,   0
v37 << 32 =    t3*16  + t4*64  ,   t5*256 + t6*1024,   0               ,   0               ,   0
v37 << 64 =    t5*256 + t6*1024,   0               ,   0               ,   0               ,   0
```
You can probably see where this is going. `_mm_packs_epi32` wrongly emphasizes "pack" instead of "s(aturation)", but it is used here to bring the multiplication's result back to 16 bits by capping it.
The other term, a constant 0, fills the 4 least significant positions of the vector. `_mm_extract_epi16(saturated_result, 7)` retrieves, or extracts, the first 16bit lane, containing `Saturate16(t1*1 + t2*4 + t3*16 + t4*64 + t5*256 + t6*1024)`.

This pattern is just a dot product. At this point doing the reversing I started having doubts about the ternary nature of this problem as powers of 4 have nothing to do with base 3.
[Spoiler] 1 trit takes 2 bits, if you replace multiplications with left shifts it's easy to see that this packs a vector into a compact form.

### Compute Scalar
What about ct_91B0? Same story but with different numbers. Instead of `t1*1 + t2*4 + t3*16 + t4*64 + t5*256 + t6*1024` you get `t1*1 + t2*3 + t3*9 + t4*27 + t5*81 + t6*243`. This is exactly how you convert a number from base 3 to base 10. In fact, the previous vector could be used to convert a number from base 4 to base 10.
Math really twists your brain, doesn't it? After I finished this problem I started questioning the nature of numbers and their representation. 

Saturation is not a concern as with 6 trits you can represent numbers up to 729, and binary-coded-ternary (BCT) representation of such numbers takes 12 bits, which is less than 16.

## 5. Ternary negation
This pattern basically confirms that this challenge uses a ternary system. It is dealing with vectors again:
```C
//xmmword_9170    xmmword 0003 0003 0003 0003 0003 0003 0003 0003h
v153 = _mm_andnot_si128(_mm_cmpeq_epi16(v152, const_0), _mm_xor_si128(v152, (__m128i)xmmword_9170));
```
Packed xor/not/and or single-integer xor/not/and is the same as there are no carries. `_mm_cmpeq_epi16` sets each lane to either 0xFFFF or 0.
Assuming each lane is a trit (i.e only last 2 bits set), xoring it with 3 does the following transformation: `0 -> 3, 1 -> 2, 2 -> 1, 3 -> 0`.

For lane `i` the operation performed is `ri = ~(v152i == 0 ? 0xFFFF : 0) & (v152i ^ 3)` or `(v152i == 0 ? 0 : 0xFFFF) & (v152i ^ 3)`.
If `v152i` is 0, the whole lane undergoes a logical and with 0 which results in 0. Otherwise LHS is 0xFFFF, a & 0xFFFF evaluates to a so the equation becomes v152i ^ 3.

0 is left unchanged, 1 becomes 2 and 2 becomes 1. This is exactly what a balanced ternary negation looks like. 3 will never be used, as the conversion at the program input shows.

## 6. Decode trit
```C
//xmmword_9170    xmmword 0003 0003 0003 0003 0003 0003 0003 0003h
//xmmword_91A0    xmmword FFFE FFFE FFFE FFFE FFFE FFFE FFFE FFFEh
_mm_srai_epi16(_mm_add_epi16( _mm_xor_si128(input, (__m128i)ct_91A0), (const __m128i *)&xmmword_9170), 1u)
```
`xmmword_9170` is usually loaded at the beginning of the program and kept in a register, I did copy-propagation on the expression so it is easier to read. IDA may show `v11` in its place.
First things first, these constants repeat every 16 bits, or once per lane. Let's treat this at the lane level, noting li the ith lane of the input.

The xor does:
```
0xE = 1110(2)
0xE ^ 0 = 0xE
0xE ^ 1 = 0xF
0xE ^ 2 = 0xC
0xE ^ 3 = 0xD
```
Note that packed or single integer xor is the same since 1 bit in the input affects only 1 bit in the output. The addition does (to each lane):
```
E (1110) + 3 = 1 0001 (1)
F (1111) + 3 = 1 0010 (2)
C (1100) + 3 =   1111 (-1) 
D (1101) + 3 = 1 0000 (0)
```
Since these are 16bit operations carries get discared, or the result overflows. Combining these two together yields a mapping
```declarative
0 -> 1
1 -> 2
2 -> -1
3 -> 0
```
`_mm_srai_epi16` is an arithmetic shift, meaning it inserts the same bit as the initial most significant, sign, bit. The shitfting amount is hard-coded to 1 byte.
The mapping becomes:
```declarative
0 -> 0
1 -> 1
2 -> 1111 (-1)
3 -> 0
```
All of this just to transform 2 into -1, or balance the ternary. It is one thing to represent the ternary -1 with "2" and another thing to do computation on the host with said "2".
I named this "decode trit" because it is similar to how characters are decoded from a file when you want to display text. I would advise against "balance ternary" because the VM operates with that already,
this just deals with storing such numbers in the host's memory.

## 7. Slice
```C
// xmmword_91C0    xmmword FFFF FFFF FFFF FFFF FFFF FFFF 0000 0000h
_mm_slli_si128(_mm_and_si128(_mm_srli_si128(v36, l), (__m128i)ct_91C0), r)
```
I can't say for sure whether this has a higher meaning as it's just a bitwise slice with spooky xmm instructions instead of `<<` and `>>` so I did not really care about it
and let it be in the codde, there's enough information to reverse manually just by renaming ct_91C0 and looking at `l` and `r`.

## 8 (non-xmm). 12bit load
I left this for last as it does not use xmmwords. After xmmwords are mostly cleared by clang or hexrays microcode optimizations this represents the main source of noise. Before:
```C
v32 = _mm_madd_epi16(a,  (__m128i)xmmword_91B0);
v33 = 3 * (__int16)_mm_extract_epi16(
             _mm_packs_epi32(
               const_0,
               _mm_add_epi32(v32, _mm_add_epi32(_mm_slli_si128(v32, 8), _mm_slli_si128(v32, 4)))),
             7)
    + 1092;
v34 = ((int)(unsigned __int8)v16[v33 >> 1] >> ((4 * v33) & 4)) | ((unsigned __int8)(v16[(v33 >> 1) + 1] & (4095 >> (8 - ((4 * v33) & 4)))) << (8 - ((4 * v33) & 4)));
```

and after:
```C
v33 = 3 * bct_to_binary(a) + 1092;
v34 = ((int)(unsigned __int8)v16[v33 >> 1] >> ((4 * v33) & 4)) | ((unsigned __int8)(v16[(v33 >> 1) + 1] & (4095 >> (8 - ((4 * v33) & 4)))) << (8 - ((4 * v33) & 4)));
```
In the binary it is always `3 * something + 12 + offset`. This sort of multiplexer reminds me of VMProtect's virtualized conditionals. That thing used binary addition instead of logical OR, but they are the same if either side is 0.
The difficulty is in proving one side is always 0. Let's start with a smaller sub-pattern: `(4 * (3 * a + 12)) & 4`:

```declarative
3 * a + 12 can be 12, 15, 18, ...
4 * (3 * a + 12) can be 48, 60, 72, and so on
(4 * (3 * a + 12)) & 4 can be 0, 4, 0, 4, ...; 4 if a is odd, 0 if even
8 - ((4 * (3 * a + 12)) & 4) can be 8, 4, 8, 4, ....
4095 >> (8 - ((4 * v33) & 4)) can be 15, 255, 15, 255, ... or 0xF, 0xFF, 0xF, 0xFF, ...
```
Now look what happens with the terms:
If a is even, `v16[v33 >> 1]` is shifted by 0 and `v16[(v33 >> 1) + 1]` undergoes a bitwise AND with 0xF. That extracts its lower 4 bits!
If a is odd, `v16[v33 >> 1]` is shifted by 4 and `v16[(v33 >> 1) + 1]` undergoes a bitwise AND with 0xFF which leaves it unchanged. That's the upper 4 bits of LHS and full RHS.
Finally, RHS is left shifted with `8 - ((4 * (3 * a + 12)) & 4)` (either 8 or 4), depending on whether LHS provides a full or half byte.

A human would write this like:
```C
uint16_t load_packed(const uint8_t * mem, const uint32_t addr)
{
    const host_index = (3 * addr) / 2;
    uint16_t p1 = mem[host_index];
    uint16_t p2 = mem[host_index + 1];
    // memory is like | a1 a2 | b1 a3 | b2 b3 | c1 c2 | ... so host indexes are 0, 1, 3, 4, ...
    
    if (addr % 2 == 0)
    {
        // tryte is 8bit aligned
        return (p2 << 8) | p1; // + p1 also works. Might have actually been the original operation, but the compiler optimized it to OR.
    }

    else
    {
        return (p2 << 4) | (p1 >> 4); // Need to get rid of the upper 2 bits of "a" which p1 contains. Again, + also works.
    }
}
```
This is how 2 values are extracted from 3 bytes! C does not have explicit zero-extends so the generated assembly might look different. As a final note, It is amazing how intricate control-flow recovery is and IDA does not even list a branch here.
Now reversing the VM's architecture is possible.

As a final note, it is amazing how intricate control-flow recovery is. IDA does not even list a branch here.