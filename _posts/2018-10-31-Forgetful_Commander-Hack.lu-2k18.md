---
title: Forgetful_Commander (Hack.lu-2k18)
date: 2018-10-31 00:00:00 +0530
categories: [CTF]
tags: [Re,exploitation, ctf]
---

So here I am again with a new post, write up of a challenge from `HackLu-2k18`. This CTF, I could only solve one challenge during the CTF. And the fun part was that, I was present there during the CTF as I was attending the `HackLu-2018` conference at Luxembourg. I will be writing about that in another post. So lets start with the challenge.

So at first I didn’t have any idea about the challenge and I tried it only because my mentor asked me to do so as it was a fun challenge.

As usual I open it in gdb, tried running it, something unusual. No inputs and no outputs. So I do the info files and run it till the address of the `.text` section and then I do `ni`, it gives a `SIGSEV`. So the binary runs till that atleast.

So I load the file in IDA. We can see that the binary tries to read what is inside `/proc/self/maps`.

It checks the path of the binary file using `sys_readlink()` in the read content. Once it reads, it does some kind of decryption on the binary. So probably the binary there should be the one which will lead us to the flag.

It maps the memory with starting address `v27+0x2000` to `v27+0x2050` and decrypts it.

So we now know the binary there should be analyzed. So at the return address `0x565621cf` we have to dump the binary from the starting and ending regions of the mapped regions.

We can use `vmmap` to get the mapped regions. I used this command to dump the binary :

`dump memory filename 0x56557000 0x56559000`

Then I open the dump in IDA to analyze the binary. I can see a function call to main.

So now we have to understand those instructions to get the flag.

### What is happening inside :

![alt text](assets/images/forgetfulcommander1.png "forgetfulcommander1")

The input is given as the command line argument. It then reads the flags and checks for the trap flag (number : `0x0100`). If it is set the value `v8` will be set to `0x40` (using `ror 0x0100 ,2`), else it will be set to `5`. So the above check can be anti-debugging check.Then there comes a simple input check.

`//v8` is the value set after the trap flag check. So I guess the value should be 5.

```python
with open('filename', 'rb') as f:
 
     file = f.read()
 
a = ''
 
for i in range(58):
 
    a += chr(ord(file[5*i+5+4104])^ord(file[i+5105]))
 
print a
```

And the flag is `flag{Just_type__Please__and_the_missles_will_be_launched.}`

And also there is a fake flag (which I realized later) which will be generated if it `TF` is set.

So overall it was a fun challenge, a kind of challenge I never tried before, I learned so many new things. So I am completely satisfied.
