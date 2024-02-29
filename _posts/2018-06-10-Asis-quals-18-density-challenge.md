---
title: Asis-Quals’18 density challenge
date: 2018-06-10 00:00:00 +0530
categories: [CTF]
tags: [Re, ctf]
---


The program took the string as an input and then added 2 random strings and added them before and after it.

### Inside the function:

It takes each letter from the string and check whether it is present in `“@$_!\”#%&'()*+,-./:;?\n”` and returns the position of the character.

If it is not null then it replaces that character with `+` and the adjacent character with `chr(position+97)`.

If it is null then it checks whether it is present in `“[\\]^{|}~`\t” `and returns the position of the character in the string.

Then it replaces the character with `+` and the next with `+` and the adjacent with `chr(position+97)`

This repeats until the string length.

Then it encodes this string in base64 and we get the output.

### Reversing the output :

The file given was b64pack. So I tried to decode it.

Command :

```
Base64 filename
```

Output :

```
O++h+b+qcASIS++e01d+c4Nd+cGoLD+cASIS+c1De4+c4H4t+cg0e5+cf0r+cls+d++gdI++j+kM+vb++fD9W+q/Cg==
```

Python script to reverse it :

```python
b64dec="O++h+b+qcASIS++e01d+c4Nd+cGoLD+cASIS+c1De4+c4H4t+cg0e5+cf0r+cl$
new=""
l=len(b64dec)
fcheck="@$_!\"#%&'()*+,-./:;?\n"
scheck="[\\]^{|}~`\t"
i=0
while(i<l-1):
        if(b64dec[i]=="+"):
                i=i+1
                if(b64dec[i]=="+"):
                        i=i+1
                        new+=scheck[ord(b64dec[i])-97]
                else:
                        new+=fcheck[ord(b64dec[i])-97]
        else:
                new+=b64dec[i]
        i=i+1
print new
```


**Output :**

```
O~$/cASIS{01d_4Nd_GoLD_ASIS_1De4_4H4t_g0e5_f0r_ls!}dI )M>b|D9W//Cg=
```

Which gives us the flag : `ASIS{01d_4Nd_GoLD_ASIS_1De4_4H4t_g0e5_f0r_ls!}`