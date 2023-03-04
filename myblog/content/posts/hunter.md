---
title: "hunter - [pwnable.kr]"
date: 2023-03-04T21:17:44+08:00
draft: false
---



Since it's recess week in my university, and as  I just got into the leaderboard of `pwnable.kr`, I decided to challenge myself and jumped into solving the least solved on the site since it released: `hunter`. This is not a writeup or anything, it's just where I blog while trying to solve it(till now I still haven't successfully solved it yet.)

I'm just gonna write about what I have tried and what is my thinking process so now headers or anything, just pure words.

First, I reversed it and did not figure out what the bug was. You know why ? Because there is actually no bug there. The only thing that can be used to exploits is the ability to overwrite the head of the `item` linked list(you will understand what I say if you also reverse it). I'm saying this because: The function that takes input of the `name` and the function that is used on weird commands use the same value to `malloc`, so we can actually trying to overwrite the head of the `item` list with the probability of 1 in 4 times:

```
size_t input_command = get_input(0x14); // weird commands
name = global_malloc(0x14); // this resides in the main function    
```

Well, if we can overwrite the head of the list, we can do something with it. At first, I thought that overwriting it would have been enough since we can definitely control the value of `bk` and `fd` of the function, however I was wrong:

```
for (int32_t loop = 1; loop < num_item; loop = (loop + 1))
	{
		current_item = current_item->Item;
	}
current_item->Item = global_malloc(0xc);
current_item->Item->bk = global_malloc(8);
```

If we overwrote the head, we would definitely control this line `current_item = current_item->Item;`. But then we cannot control the next 2 lines :). This is the place I suffer :|. I don't know what to do next, I thought about using the "heap spray" from spawning a monster to use as a value and also `money` is in our control, but after drafting it out on paper, I believed that overwriting it was not enough. I am missing something.

