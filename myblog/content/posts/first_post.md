---
title: "[Dreamhack.io] Wishlist"
date: 2023-02-15T14:06:12+08:00
draft: true
---

For the first post of my blog, I will do a partial writeup (some analysis and ideas) on a nice heap grooming challenge on [dreamhack.io](https://dreamhack.io/), which at the time of writing, has only 12 solves. This is probablly go against the policy of the site but the way I leak the address was quite cool (however at the end I figured out that I was an idiot)[^1]. So let's start.

## 1. _Overview_:
For this challenge, we were given a **binary** along with **Dockerfille**, **source code** and a **llbc++**. This means we are dealing with a  I always enjoy challenge that give you source code cuz I'm suck at RE. As usual, I will run some basics file checking on the binary. 
`checksec` output: 
	![[Pasted image 20230216014512.png]]

So `Full RELRO`, means we cannot overwrite anything on the GOT table.

Run the challenge, we can see that it prompt us with a menu-base challenge, so most likely this will be a heap challenge.
![[Pasted image 20230217194957.png]]
Also if we see the `Dockerfile` and pull libc on it, we can see that its libc version is 2.31, which means we are dealing with much more secure version than the older one like 2.27 or 2.23. However, this version still allows us to overwrite hooks like `__free_hook` and `__malloc_hook`(unlike current libc, where this technique has been decapriated), which is exactly what we gonna do in this challenge.

## 2. _Source code analysis and bug_:
We were given the source code so it will be easier for us to analyze the binary.  The source code was written in a very clear way and that is more pleasing. Let's dive into the code.

The `main` function call three other function: `add_wishlist`, `show_wishlist` and `delete_wishlist` and `send_wishlist`. `send_wishlist` is useless so we will not talk about it here. Let's talk about the  remaining.
As any other classic heap challenges, `add_wishlist` let us add an item to the list, `show_wishlist` let us view the content of the list, and `delete_wishlist` let us delete an item in the list.  This is all classic heap features. In `show_wishlist`, if the name is not yet input, it will ask us to input our name with `NAME_SIZE = 16`
After looking at this source code for sometime, I couldn't find any bug. But then I remember that since this is a heap challenges, and in heap challenges, the bug usually resides in the `input ` function. So I checked for the `input()`, and found our sweet spot:

```
void input(char *buf, int size) {
    std::cin.getline(buf, size+1);
}
```
See that `size + 1`? If you check back in `set_name`,  you can see that it calls `input`, thus  let the user input 17 bytes instead of 16. That is `off-by-one`. 

Also, we have a `win` function, thus making our life a little bit easier.

## 3.  _Arbitrary write (nearly) and the leak_
Let's try to see what can we do with off-by-one. If we create 2 items, then use menu 2 to input 16 bytes, it will get a one-byte overflow. Let's check.
If we enter normally, we will get something like this: 
```
======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 1

[+] Add Wishlist

Wish item > aaaabbbb

======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 1

[+] Add Wishlist

Wish item > asdfadsf

======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 2

Your Name > asdfasdf

[+] asdfasdf's Wish List

------------------------

 No. 1

  Item. aaaabbbb

 No. 2

  Item. asdfadsf

------------------------
```
However, if we enter like mentioned, see what happens:
```
======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 1

[+] Add Wishlist

Wish item > aaaaaaaaaaaaaaaaaaaaaaaa

======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 1

[+] Add Wishlist

Wish item > bbbbbbbbbbbbbbbbbbbbbbbb

======== [MENU] ========

   1. Add Wish List     

   2. My Wish List      

   3. Delete Wish List  

   4. Send Wish List!   

========================

> 2

Your Name > cccccccccccccccc

[+] cccccccccccccccc's Wish List

------------------------

 No. 5983606249452696417

  Item. incerely,





 No. 7161677110969590627

  Item. cccccccc

 No. 4300992

  Item. 1

 No. 7016996765293437281

  Item. aaaaaaaa

 No. 1

  Item. aaaaaaaaaaaaaaaaaaaaaaaa

 No. 2

  Item. bbbbbbbbbbbbbbbbbbbbbbbb

```

We might have overwrite the LSB of the begin pointer of the `wishlist`, and gdb confirms it:

```
0x41a020	0x6363636363636363	0x6363636363636363	cccccccccccccccc

0x41a030	0x000000000041a000	0x000000000041a0c0	..A.......A.....

0x41a040	0x000000000041a0c0	0x0000000000000031	..A.....1.......

0x41a050	0x0000000000000000	0x0000000000408010	..........@.....	 <-- tcachebins[0x30][0/1]

0x41a060	0x6161616161616161	0x6161616161616161	aaaaaaaaaaaaaaaa

0x41a070	0x0000000000000000	0x0000000000000051	........Q.......

0x41a080	0x0000000000000001	0x6161616161616161	........aaaaaaaa

0x41a090	0x6161616161616161	0x6161616161616161	aaaaaaaaaaaaaaaa

0x41a0a0	0x0000000000000002	0x6262626262626262	........bbbbbbbb

0x41a0b0	0x6262626262626262	0x6262626262626262	bbbbbbbbbbbbbbbb

0x41a0c0	0x0000000000000000	0x000000000000ef41	........A.......	 <-- Top chunk
```

We can see that `0x41a030	0x000000000041a000`, with its LSB is null and point upwards when it supposed to point to be `0x41a080`. So with this, we can almost get an arbitrary write on the heap by grooming it in the right way. 

So what can we do with arbitrary write on the heap (almost) ? Remember that, the list here is a `vector`, so if we `erase` an element in the list, it will move all the element above the erased element down in memory. With this, we can actually groom the heap and overwrite the `begin` and `end` pointer of the vector. Since this is libc 2.31, overwrite `__free_hook` is still a thing, so we can get that as an idea. However, to be able to do that, we still need a leak. This binary was written purely in C++, so we can (maybe) only leak C++ function address. However, in practice, the relative address between libraries are the same, so if we leak one library, we are actually having all other libraries address.

Now the fun part. How do we leak an address ?  Well,  