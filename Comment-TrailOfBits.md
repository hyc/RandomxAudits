Comments from @tevador on version 1 of the report. The final report uploaded here is version 3,
and already incorporates a number of the changes noted here.

## 1. Single AES rounds used in AesGenerator

In general, I agree that single-round AES can produce biased random data.

We have implemented a mitigation in PR #46: https://github.com/tevador/RandomX/pull/46 by using 4 AES rounds for program generation.

We cannot use more than 1 AES round for Scratchpad initialization because it would increase runtime overhead - see https://github.com/tevador/RandomX/blob/master/doc/design.md#fast-mode---program-execution
Current overhead is around 7.5% of 'not useful' work, which would increase to 10% if we used 2 AES rounds for Scratchpad initialization. Additionally, there is no benefit from having unbiased Scratchpad as the data is overwritten during VM execution.

The exploit scenario is unrealistic. Using 1 AES round causes insufficient mixing only in 2 adjacent 16-byte blocks, which is nowhere near the level of control needed to zero an output bit. The real effect of using just 1 AES round for program generation is having slightly correlated bits in some adjacent instructions, which slightly decreases the randomness of the program. To increase the security margin, we have decide to use 4 AES rounds for program generation (but keep 1 round for Scratchpad initialization).

## 2. Insufficient Testing and Validation of VM Correctness

I agree that we should provide at least test vectors for portions of the VM.

## 3. RandomX configurable parameters are brittle

I agree that some parameters have a much larger effect than others. Providing guidelines for safe changes of parameter values is a good idea.

## Appendix B. Code Quality

1. All code in the src/tests directory should be excluded from the audit as it's not shipped in the RandomX library and will not be part of Monero.

2. Agreed.

3. I guess that's a matter of personal coding preference, but I agree that there are benefits to using strongly typed enums.

## C. Randomness Analysis

While the cryptanalysis of Blake2b is certainly out of the scope of the audit, I recommend citing some of the many papers that have been published on this topic. For example: https://eprint.iacr.org/2016/827.pdf

Quote from the abstract:
"We prove that BLAKE2’s compression function is indifferentiable from a random function in a weakly ideal cipher model [...] This implies that there are no generic attacks against any of the modes that BLAKE2 uses."

Figure 2 shows a zoomed in view of the distribution of the byte values. Trail of Bits perhaps inadvertently proved that AesGenerator produces an ideal output distribution. A perfect generator will follow the Poisson distribution and the standard deviation of such distribution is equal to the square root of the mean value.

√(134.2 million) = 11.5 thousand

(The listed value of 23 thousand is actually equal to 2 standard deviations.)

Source:
https://en.wikipedia.org/wiki/Poisson_distribution#Other_applications_in_science

I would also like to point out that a uniform distribution of byte values is not a sufficient condition for a good PRNG. For example, the sequence 0, 1, 2, ..., 254, 255, 0, 1, 2, ... will yield a perfect distribution despite being an awful random sequence.

## D. Bias in SuperscalarHash

SuperscalarHash is not a linear congruential generator! LCG is only used to initialize the initial value of the r0 register, see chapter 7.3 of the specification: https://github.com/tevador/RandomX/blob/master/doc/specs.md#73-dataset-block-generation

The uniqueness of bits 3-53 is *not* a requirement for unique Dataset items. In fact, even if all registers r0-r7 were initialized to the same value, it would probably still give an uncompressible Dataset. We have chosen to modify the input parameters to have unique values of bits 3-53 as an additional security measure. A Dataset item is generated from 8 chained instances of SuperscalarHash (interleaved with random Cache accesses), which is enough to mix even an imperfect initial state in most cases.

Updating SuperscalarHash to achieve full avalanche is an impossible task because it's a randomly generated function and its objective is to execute as many ALU instructions as possible in 170 cycles using the fact that CPUs are superscalar.

To get an idea what a SuperscalarHash instance looks like, see the assembly file appended below (this file was generated using the "code-generator" executable).

## E. Parameter Selection

The recommendations listed in this Appendix generally make sense.

A small correction: there is a hard constraint in the code (common.hpp:55)

RANDOMX_JUMP_BITS + RANDOMX_JUMP_OFFSET <= 16

So RANDOMX_JUMP_OFFSET cannot be in the range 0-63. This is related to the fact that the CBRANCH instruction includes additional 0-15 bit offset and immediate values are limited to 32 bits.

## F. Recommended Parameters for Arweave

It should be noted that the changes in Table 2 will reduce proof verification speed approximately by a factor of 2-3. It's up to Arweave if they are comfortable with this.

 ---

superscalar1.asm
````
imul r14, r12
imul r12, r11
imul r8, r15
ror r13, 30
ror r15, 24
add r13, 1996107270
mov rax, r9
mul r15
mov r9, rdx
mov rax, -7159297672498682988
imul r11, rax
xor r14, 1477786261
ror r13, 53
ror r14, 46
add r12, 680756024
ror r8, 61
imul r13, r15
imul r15, r8
imul r8, r14
lea r14, [r14+r12*8]
xor r12, r10
add r14, 1673483064
sub r11, r9
xor r10, r14
sub r9, r12
xor r13, -1511045831
sub r10, r13
mov rax, r12
mul r11
mov r12, rdx
mov rax, -5136144375103002627
imul r14, rax
imul r9, r11
imul r11, r13
ror r13, 33
xor r8, r10
add r13, 274430509
sub r13, r9
xor r8, r15
sub r9, r8
xor r15, -1290951162
xor r8, r13
sub r14, r8
imul r12, r8
imul r13, r10
imul r10, r14
lea r15, [r15+r8*1]
ror r14, 54
xor r9, 1467689628
mov rax, r8
imul r14
mov r8, rdx
mov rax, -8219025209819642291
imul r13, rax
xor r11, -899761052
mov rax, r15
mul r10
mov r15, rdx
mov rax, -8916774622625733947
imul r10, rax
xor r14, 1875778603
lea r9, [r9+r11*8]
lea r14, [r14+r12*4]
add r9, 50960751
xor r12, r11
imul r11, r8
imul r12, r9
imul r14, r10
lea r8, [r8+r9*1]
sub r9, r13
xor r13, 493065189
sub r15, r9
mov rax, r8
mul r13
mov r8, rdx
mov rax, -5640020751233637551
imul r11, rax
add r13, -27004973
xor r9, r13
xor r12, r13
add r10, 287441670
sub r12, r10
mov rax, r15
imul r10
mov r15, rdx
mov rax, -6817616021590279509
imul r10, rax
imul r13, r9
imul r12, r13
ror r9, 61
ror r8, 41
add r9, -557459299
mov rax, r14
mul r13
mov r14, rdx
mov rax, -7653138004408218578
imul r9, rax
xor r8, -1780101650
mov rax, r11
imul r10
mov r11, rdx
mov rax, -6151655414755656770
imul r13, rax
add r10, 1603447072
mov rax, r8
imul r8
mov r8, rdx
mov rax, -4526723716994567935
imul r10, rax
add r12, 1426309469
mov rax, r15
imul r14
mov r15, rdx
mov rax, -8802239477848828355
imul r12, rax
add r14, 1844749410
lea r14, [r14+r9*8]
ror r9, 27
add r9, 1167468255
xor r13, r9
imul r8, r11
imul r11, r9
imul r15, r13
lea r9, [r9+r14*8]
sub r10, r9
xor r14, 403556291
xor r9, r14
mov rax, r13
mul r14
mov r13, rdx
mov rax, -5768261838873006214
imul r10, rax
add r12, -753864129
lea r14, [r14+r9*2]
ror r8, 27
xor r12, 931777546
lea r11, [r11+r9*1]
imul r12, r15
imul r14, r8
imul r8, r10
lea r9, [r9+r15*1]
xor r15, r11
add r11, 90017237
xor r15, r9
xor r10, r13
xor r15, -1691995274
xor r10, r11
sub r13, r15
mov rax, r9
mul r13
mov r9, rdx
mov rax, -6803305357194856743
imul r10, rax
imul r15, r12
imul r11, r13
lea r14, [r14+r12*4]
lea r14, [r14+r13*8]
add r12, 279874214
mov rax, r13
mul r10
mov r13, rdx
mov rax, -6796432362329642003
imul r8, rax
xor r14, 336376139
lea r12, [r12+r14*4]
xor r10, r14
xor r12, -1201327458
sub r9, r14
mov rax, r14
mul r9
mov r14, rdx
mov rax, -7241823674429888885
imul r9, rax
imul r8, r10
imul r10, r12
lea r11, [r11+r15*2]
ror r12, 20
xor r15, 1016612505
lea r12, [r12+r13*2]
ror r13, 29
xor r11, -1077286391
mov rax, r15
mul r11
mov r15, rdx
mov rax, -284390263556890112
imul r13, rax
imul r14, r12
imul r12, r9
ror r9, 40
xor r11, r8
add r8, -196297603
xor r9, r11
mov rax, r8
mul r9
mov r8, rdx
mov rax, -5758060418923647391
imul r10, rax
add r11, 678319733
ror r9, 22
xor r11, r9
xor r11, 1014537041
sub r13, r14
xor r12, r11
imul r15, r9
imul r11, r12
imul r10, r8
ror r14, 6
add r9, 885391204
xor r14, r12
xor r12, r8
xor r9, r14
lea r15, [r15+r12*8]
xor r12, 1659234016
ror r14, 42
imul r13, r8
imul r12, r8
imul r9, r8
ror r15, 63
add r11, -892291126
sub r14, r10
xor r15, r13
xor r10, r14
lea r11, [r11+r14*8]
add r11, -115906238
lea r10, [r10+r12*4]
imul r14, r8
imul r15, r11
imul r8, r10
lea r9, [r9+r13*8]
xor r12, r13
xor r12, 1101928357
sub r13, r9
mov rax, r10
imul r14
mov r10, rdx
mov rax, -7297318945031782625
imul r9, rax
add r15, 1114349141
ror r11, 9
lea r11, [r11+r14*1]
add r12, 485945653
ror r15, 32
imul r12, r13
imul r13, r14
imul r9, r14
ror r14, 60
lea r8, [r8+r14*8]
add r11, 625400140
lea r11, [r11+r15*2]
lea r10, [r10+r8*2]
xor r15, -1028306861
xor r11, r12
imul r8, r13
imul r15, r14
imul r14, r10
lea r12, [r12+r13*2]
ror r13, 49
xor r9, -1459844162
ror r11, 7
lea r12, [r12+r9*2]
add r12, -926122778
sub r10, r9
imul r11, r8
imul r9, r13
imul r13, r10
ror r12, 42
ror r15, 57
add r12, 41050280
lea r10, [r10+r14*8]
add r15, 1808413685
sub r14, r10
sub r8, r10
mov rax, r11
mul r9
mov r11, rdx
mov rax, -7536171404844214491
imul r13, rax
imul r12, r8
imul r14, r15
ror r9, 56
xor r15, 307644266
sub r9, r10
sub r10, r8
sub r15, r9
xor r8, -1681009005
xor r9, r13
xor r8, r10
xor r13, r10
imul r15, r13
imul r11, r12
imul r13, r10
ror r10, 49
xor r12, -1141629931
sub r9, r14
xor r10, r12
xor r10, r8
lea r8, [r8+r9*8]
xor r15, 1075445833
ror r12, 14
imul r8, r10
imul r15, r10
imul r10, r12
ror r14, 10
lea r12, [r12+r11*2]
add r14, -654798400
mov rax, r9
mul r14
mov r9, rdx
mov rax, -8746457810794811539
imul r13, rax
add r12, -152049626
ror r12, 54
xor r8, -1222368824
xor r14, r11
sub r15, r11
xor r11, r12
imul r8, r10
imul r13, r11
imul r9, r11
lea r11, [r11+r14*4]
lea r14, [r14+r10*8]
xor r12, -597683788
lea r12, [r12+r14*1]
lea r11, [r11+r15*1]
add r10, -2106052380
sub r10, r15
imul r15, r8
imul r11, r14
imul r10, r14
lea r14, [r14+r12*1]
sub r8, r12
add r8, -1751291214
xor r9, r13
sub r12, r8
xor r15, 616087284
xor r12, r13
sub r14, r8
mov rax, r13
mul r13
mov r13, rdx
mov rax, -7179276621804228827
imul r11, rax
imul r12, r8
imul r9, r8
ror r15, 14
lea r8, [r8+r10*4]
add r15, -1016585450
mov rax, r14
mul r14
mov r14, rdx
mov rax, -2893566455426960199
imul r8, rax
xor r10, -764508024
lea r11, [r11+r15*2]
xor r15, -181908202
sub r15, r11
sub r11, r13
mov rax, r13
mul r9
mov r13, rdx
mov rax, -3281234635128312888
imul r9, rax
imul r11, r14
imul r15, r14
ror r12, 58
ror r10, 62
xor r14, -393927199
lea r8, [r8+r12*4]
ror r14, 30
add r8, -679892071
sub r12, r8
imul r14, r13
imul r9, r8
imul r13, r12
lea r10, [r10+r8*1]
lea r11, [r11+r8*8]
add r12, -1878504678
lea r12, [r12+r11*1]
xor r8, r10
xor r14, -1953783120
sub r10, r15
mov rax, r11
imul r15
mov r11, rdx
mov rax, -5098819608079635959
imul r10, rax
imul r12, r13
imul r8, r9
lea r14, [r14+r9*4]
xor r14, 129149443
xor r15, r13
sub r14, r13
mov rax, r9
imul r14
mov r9, rdx
mov rax, -9074209399041313697
imul r15, rax
add r14, 1625169389
lea r10, [r10+r13*4]
xor r14, r13
add r13, -1912443888
sub r11, r13
sub r12, r8
imul r14, r10
imul r11, r12
imul r15, r9
ror r10, 54
ror r8, 9
add r12, 471539138
ror r9, 3
lea r10, [r10+r12*4]
add r14, 540905597
mov rax, r13
mul r12
mov r13, rdx
mov rax, -2664623545871167684
imul r11, rax
imul r12, r8
imul r9, r14
ror r14, 58
sub r14, r15
add r8, 526756952
sub r15, r8
xor r14, r8
ror r12, 45
xor r15, 1767199763
mov rax, r10
mul r13
mov r10, rdx
mov rax, -5677575203989817534
imul r12, rax
imul r14, r13
imul r8, r15
ror r11, 42
xor r13, r9
add r9, -1520137133
sub r15, r9
mov rax, r11
mul r12
mov r11, rdx
mov rax, -5733649157757831092
imul r14, rax
xor r15, -39409424
ror r13, 12
sub r15, r12
add r13, 479820163
xor r10, r12
mov rax, r9
imul r12
mov r9, rdx
mov rax, -9167904622037064427
imul r10, rax
imul r12, r15
imul r13, r8
lea r8, [r8+r15*1]
lea r14, [r14+r11*2]
add r15, -1881578160
xor r8, r15
lea r8, [r8+r15*1]
xor r15, -930303208
sub r11, r8
imul r8, r10
imul r14, r10
imul r15, r9
ror r10, 28
lea r9, [r9+r11*2]
xor r10, 100461729
lea r11, [r11+r10*8]
xor r12, 555183652
xor r12, r13
xor r11, r13
xor r10, r8
imul r11, r12
imul r12, r15
imul r10, r15
ror r13, 63
add r9, 1584769709
xor r9, r8
sub r14, r8
xor r13, r8
lea r14, [r14+r15*1]
xor r9, -820105936
lea r8, [r8+r11*1]
imul r14, r13
imul r9, r11
imul r13, r15
lea r12, [r12+r11*4]
add r15, -1403272226
sub r10, r8
sub r15, r11
sub r12, r14
ror r10, 46
xor r15, 1639228841
lea r10, [r10+r14*1]
imul r15, r8
imul r8, r12
imul r12, r11
lea r9, [r9+r14*4]
ror r14, 2
xor r10, -1663737580
ror r11, 23
sub r10, r13
add r10, -1766892769
xor r15, r9
xor r9, r14
imul r9, r13
imul r10, r12
imul r14, r8
ror r8, 59
ror r15, 25
add r11, 2094521477
lea r12, [r12+r8*2]
add r13, 917053015
sub r15, r12
xor r11, r12
xor r9, r12
imul r8, r12
imul r15, r9
imul r9, r13
lea r10, [r10+r13*4]
lea r12, [r12+r14*8]
xor r13, -1700010387
sub r14, r11
xor r14, 810334986
xor r12, r8
xor r11, r10
mov rax, r10
imul r14
mov r10, rdx
mov rax, -7271860892641319345
imul r11, rax
imul r13, r8
imul r12, r14
````
