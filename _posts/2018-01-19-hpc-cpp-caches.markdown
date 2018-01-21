---
layout: post
title: A No-Frills Introduction To Writing Highly Performing C++ Code Part 1 - CPU caches
date: 2018-01-19 00:00:00 +0300
img: cpu-images-min.jpg
tags: [performance, C++, tutorial]
---
As you know C++ development is all about performance. Well not exactly,
C++ is just a language that gives you great control over hardware and low-level
resource management. This can include networking, memory and what most
people care about - code execution time.

When you want a better performing application there are 3 levels on
which you can improve.

1. High-level system design - this includes how your
components communicate with each other. (Having 10 levels of indirection
although correct might not be the best performing architecture)

2. Algorithm level - pretty self-explanatory O(n) is better than O(n^2)
 In this article I will give you

3. Micro optimizations - same architecture, same algorithm but better
hardware performance(e.g. less CPU instructions, less memory stalls)

In this article I will focus on micro optimizations for memory access.

The first thing that can grately improve your performance is utilizing
CPU caches. Modern CPUs have 2 or 3 levels of cache memory, until they
reach to the main(RAM) memory, whenever you need some piece of data.


| Memory Type | Access Time |
|-------------|-------------|
| L1 Cache    | 0.5ns       |
| L2 Cache    | 7ns         |
| RAM         | 100ns       |

As you can see using caches seems very desirable, but how do you do it?

Caches are just faster than RAM memory which stores the same data that you
have in RAM but the access is magnitudes faster.
Whenever you access a memory address, some fixed amount of memory is pulled from
RAM to the cache (called a cache line), for example 64 bytes. When you access
address X, the memory between X and X+64, will be cached. Since caches
are usually a few KB of size the memory you read won't stay in the cache forever,
but the duration depends on the **eviction** policy, I won't go into detail here about them.
Memory caching works on two simple principles:
* Temporal locality - if you are using memory address **now** you will probably need it again **soon**
* Spatial locality - if you used **this** memory address, you will probably need those **close** to it as well

So to improve your code you better structure it in a way, which uses same data for multiple computations at once,
or lay out needed data in a way that when you access data Y, the memory around it will be read as well.

So given the question "In what collection would you store N numbers if you want to sum them,
linked list(std::list) or array(std::array/vector)?" what is your intuition, what collection would you choose and why?
Most developers will tell you it doesn't matter you have to traverse
the whole collection, so both choices are still O(n). Although this is correct
the actual time on a real machine won't be close. The reason is that
array are just a chunk of sequential memory while linked lists are usually implemented
with pointer to another cell in memory. Let's play out a simple example.

Lets sum 64 ints, the usual implementation with array looks like this.
```c++
int sum = 0;
const int arr[64] = {...}
for (int i = 0; i < 64; ++i)
{
    int += arr[i];
}
```
The memory access patter will be like this:
* arr[0] - **cache miss** 100ns to get that data from RAM, but the size of
int is usually 4 bytes if we have 64 byte cache line. This means that
we've effectively pulled arr[1] to arr[15] in cache.
summing arr[1..15] will cost 15 * 0.5ns.

* When **i=16** we won't have this in cache memory so it will be a
**cache miss** - 100ns and then arr[17..31] - 0.5ns.
(Disclaimer: We might have it if the hardware prefetcher is smart but this is another topic.)

* This pattern will continue so the total cost for memory access
will be, 4 * 100ns + 60 * 0.5ns = 430ns.

Similar implementation but with linked list will look like this
```c++
struct Node
{
    int Num;
    Node* Next = nullptr;
}
/*list creation with start of the list **head***/
Node* currNode = head;
int sum = 0;
while (currNode)
{
    sum += currNode->Num;
    currNode = currNode->Next;
}
```
* When we do currNode->Num we chase the **currNode** pointer to read the memory there
(we wan't the **Num**), this is 100ns, this caches for future use **Num** and **Next**,
we then access **Next** through currNode but it is in cache so this is just 0.5ns

* The next time we access currNode we need to access different memory location on totally unrelated address,
which means **cache miss** so again 100ns and so on until we are done with the list.

** Every loop iteration we have a cache miss so if we were again summing 64 numbers,
the total cost for memory access will be 64 * 100ns + 64 * 0.5ns = 6432ns,
obviouslythis is way more than those 430ns for the array summing.

This should convince you that using CPU caches can increase your code performance
by far, but how can you achieve it?

Try to use sequentially laid out in memory data structures or partially sequential:
1. std::array - fully sequential
2. std::vector - fully sequential
3. std::unordered_(map/set) - partially sequential
4. std::deque - partially sequential

Avoid pointer chasing dependant structures:
1. std::list - or any kind of list
2. std::set/map - they are implementd in terms of trees
3. **trees** - they are just a generalized version of lists.

This should give you a basic understanding of CPU caches and how you can
increase you code performance. Using sequential data structures is just
the basics, there are more advanced topics about cache utilization, for example
avoiding cache thrashing, false sharing, data-oriented desing etc.