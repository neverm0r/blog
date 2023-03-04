---
title: "[pwnable.kr] elf"
date: 2023-02-24T12:56:37+08:00
draft: true
author: neverm0r
---

Recently,  I solved the `elf` challenge on pwnable.kr, which helped me get to the leaderboard of it.  I realized I learned a lot after solving it so I decide I will do a partial writeup on the challenge as well as note back what  I have learnt. Let's get right into it.

## 1. Overview

When you ssh to the  pwnable.kr server (which is the one thing I like from pwnable.kr since it makes the exploits work sbetter since we are able to do it kind of locally), we are given two `python` files, `elf.py` and `gen.py`. 

`elf.py`:

```
libc = CDLL('libc.so.6')
flag = CDLL('./libflag.so')
...
for i in xrange(25):
        sys.stdout.write('addr?:')
        sys.stdout.flush()
        addr = int(raw_input(), 16)
        libc.write(1, c_char_p(addr), 32)
```

`elf.py` gives us 25 times to read 32 bytes from the output.

`gen.py`:

```
code = '''
        #define _GNU_SOURCE
        #include <stdio.h>
        '''

        for i in xrange( random.randrange(10000) ):
                code += 'void not_my_flag{0}(){{printf("not a flag!\\n");}}\n'.format(i)
        FLAG = ''
        with open('flag', 'r') as f:
                FLAG = f.read().strip()

        code += 'void yes_ur_flag(){{ char flag[]={{"{0}"}}; puts(flag);}}\n'.format(FLAG)

        for i in xrange( random.randrange(10000) ):
                code += 'void not_ur_flag{0}(){{printf("not a flag!\\n");}}\n'.format(i)
        with open('./libflag.c', 'w') as f:
                f.write(code)

        os.system('gcc -o ./libflag.so ./libflag.c -fPIC -shared -ldl 2> /dev/null')	
```

This script generate an object file, which contains a random number of `not_my_flag` and `not_ur_flag` functions, but have a `yes_ur_flag` functions, which we need to run in order to get the flag. When I first look at this challenge, I thought this challenge was too were since we only have only 2 python scripts :|. So after an hour of thinking, I decided to run `checksec` on python, and found out that it was `No PIE`.

```
elf@pwnable:~$ checksec /usr/bin/python
[*] '/usr/bin/python'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
    FORTIFY:  Enabled
```

This must be intentional. Remember we can input an address in hex then it prints out the content of it? We can actually use this to leak libc since python it's `No PIE` (if python has `PIE` I don't think this chall is pwnable). 

However, even with a leak, what to do next ? If copy both scripts into `/tmp` to debug, you will realize that the end address of `libflag.so` and the start address of `libc` is actually differ with  a constant offset:

```
0x00007f905bad4000 0x00007f905bbe4000 r-xp      /tmp/hshs/libflag.so
0x00007f905bbe4000 0x00007f905bde4000 ---p      /tmp/hshs/libflag.so
0x00007f905bde4000 0x00007f905bde5000 r--p      /tmp/hshs/libflag.so
0x00007f905bde5000 0x00007f905bde6000 rw-p      /tmp/hshs/libflag.so
...
0x00007f905d35d000 0x00007f905d35e000 r--p      /lib/x86_64-linux-gnu/libdl-2.23.so
0x00007f905d35e000 0x00007f905d35f000 rw-p      /lib/x86_64-linux-gnu/libdl-2.23.so
0x00007f905d35f000 0x00007f905d51f000 r-xp      /lib/x86_64-linux-gnu/libc-2.23.so
gdb-peda$ p/x 0x00007f905d35f000 - 0x00007f905bbe4000
$1 = 0x177b000	
```

Just run it a few times, then you can realize that they always apart with a constant offset `0x177b000` 





