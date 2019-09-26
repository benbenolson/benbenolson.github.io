---
layout:     post
title:      Memory Alignment Bug
date:       2019-09-13
summary:    A rare bug, related to allocating aligned memory, that took a while to debug.
categories: jekyll
---

## The Issue

For the past few years, I've been using `jemalloc` to allocate memory in my
experiments, partly because of the usefulness of arena allocation. In using
`jemalloc`, when you allocate memory to an arena, an actual allocation may or
may not happen, depending on if that arena requires more memory or not. Because
these arenas allocate memory in multiple-megabyte chunks (or `extents`, as
they're called in `jemalloc`), this allocation does not necessarily happen each
time you allocate to the arena. Instead, `jemalloc` allocates an extent only
when needed, then holds onto that extent until the arena is freed or until some
other heuristic determines that it should be freed.

However, for my research, I need to be able to bind all arena memory to a
specific NUMA node; that is, for every extent allocation, I need to call
`mbind` on that range of addresses. To do something like this, `jemalloc`
provides the `extent_hook` interface. Briefly, this feature allows you to
define a set of function pointers that will be used for all memory operations
in the arena: allocating, deallocating, committing, etc. Notably, we define
a function called `sa_alloc`, which allocates a new extent to an arena.

`sa_alloc` accepts quite a few arguments, including the size of the extent that
it should allocate (`size`), and the number of bytes that it should align that
allocation by (`alignment`). The first argument is handled easily: simply call
`mmap` and pass it the correct size. The second argument is a little trickier,
but this is how the original code did it. I'll remove the error handling and
other distracting elements. This code was also not originally written by me, so
I've added my own comments:

{% highlight C %}

  /* Do  the initial allocation */
	ret = mmap(new_addr, size, PROT_READ | PROT_WRITE, mmflags, sa->fd, sa->size);

  /* Check if the allocation meets the alignment */
	if (alignment == 0 || ((uintptr_t) ret)%alignment == 0) {
		goto success;
	}

  /* If it's not aligned properly, unmap and try again with a larger size */
	munmap(ret, size);
	size += alignment;
	ret = mmap(NULL, size, PROT_READ | PROT_WRITE, mmflags, sa->fd, sa->size);

  /* Chop off and unmap the excess */
	n = (uintptr_t) ret;
	m = n + alignment - (n%alignment);
	munmap(ret, m-n);
	ret = (void *) m;

  /* Finally, call 'mbind' on the new extent */
success:
	mbind(ret, size, mpol, nodemaskp, maxnode, MPOL_MF_MOVE) < 0);
{% endhighlight %}

For several years, this worked for a variety of applications and never threw
any errors.  However, only recently and in certain situations, it began
throwing extremely rare errors for me in one particular application: AMG.
Specifically, the final `mbind` call would return `Bad address`, the function
would fail, and the runtime library would fail to allocate to an arena. One
other thing that I should note is that while `perror` prints out `Bad address`,
it's possible that the error is caused by some of the other arguments. Poking
through the `do_mbind` function in the Linux kernel, it seems as if there are a
great deal of other things that could cause that same error, being `EFAULT`.

## The Search

To begin with, I didn't even consider the possibility that this code could be
flawed.  After all, it succeeded for several years before this, and I'd never
seen it throw an error in the past. Nearly every day, I would run experiments
that required this block of code, yet only now was it failing. After quickly
printing out the arguments to `mbind` (they looked entirely ordinary), I
started my search with what I considered to be more error-prone parts of the
codebase.

Being an error that happens every so often, my first though was a race
condition: something, I thought, must be interfering with this allocation in
some way, perhaps changing some of the heap-allocated structures from which I
get some of the arguments to `mbind`. However, all of these variables are
protected by an arena-wide mutex, so it's not possible for two threads to
allocate to a single arena simultaneously.

The second possibility that I considered is that one of the other threads in
the runtime library was causing it: sometimes, I might run the application with
a profiling thread or a `jemalloc` background thread disabled, and it would
succeed. Because of this, I theorized, it might be that thread in the runtime
library that's messing with the heap in some way. However, after running it
several more times, it would eventually fail with those threads disabled.

The last thing that I considered was that something had gone awry with onlining
some of the memory nodes. As in my previous post, I had some difficulty with
some of the blocks of memory on my system being `ZONE_MOVABLE`. This being a
recent problem, coupled with the fact that the error failed more consistently
when memory spilled onto other NUMA nodes, convinced me that `mbind` wasn't
able to allocate to certain regions of one of the NUMA nodes. That, however,
was quickly debunked by simply binding to different nodes, which resulted in
the same issue.

## The Solution

Finally, I took a harder look at the alignment code: were there rare situations
in which it could fail, depending on some race condition? Printing out `m`,
`n`, `ret`, etc., I suddenly realized the issue: after failing to get the
correct alignment, the code adds `alignment` to the `size`. Then, after
adjusting the pointer and unmapping the excess, `size` remains the same. This
then gets passed to `mbind`, which is looking for a block of memory of size
`size`. However, because part of this allocated block was subsequently unmapped,
the size needs to be updated to reflect that.

Now, why did it take so long for this issue to finally crop up? How could I run
experiments for several years without experiencing this issue even once? I think that
in order for this issue to actually occur, a perfect storm of conditions must be true:

  1. The application must request a large amount of memory aligned by a size
     larger than the page size.
  2. The application must be extremely heavily threaded.
  3. The application must use lots of allocations and deallocations from those
     threads.

The vast majority of applications that I run don't necessarily need aligned
memory, or require it to be aligned by pages (which is always true), so that
code very rarely even runs.  When it does run, however, I suspect that the
majority of the time, the `munmap` might not actually do anything: that is,
`mmap` will give more memory than is required, or the small size of the
allocation means that the `munmap` doesn't see a reason to actually deallocate
a particular set of pages. When it does, though, it's possible that the
application itself still holds the memory that was originall allocated by the
second `mmap`, and thus `mbind` won't have any trouble binding that memory to
another node. I think, in order for the issue to crop up, an application must
have a large number of threads constantly allocating and deallocating from
various arenas, causing some fragmentation in the heap and eventually causing
holes which have a chance of not being allocated by other threads in the
application.
