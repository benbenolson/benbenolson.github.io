---
layout:     post
title:      Memory Zone Subtleties
date:       2019-09-13
summary:    How to use AEP DIMMs as a NUMA node, and fix common issues.
categories: jekyll
---

## Introduction

This week, I had a particularly subtle issue that took me several days to find
and fix.  Although simple to fix, diagnosing the problem was difficult. This
post involves a little tutorial on how to use Intel's AEP DIMMs, as well as
descriptions of some issues that I encountered while using them.

I'm on a system with some Intel® Optane™ Persistent Memory DIMMs, which contain
large capacities of non-volatile memory. From here on out, I'll refer to them
as "AEP", which stands for "Apache Pass", the memory technology that they use
under the hood. The three most common ways of making this memory available to
the operating system are:

  1. 2LM mode, enabled in the BIOS, which uses a number of DDR4 DIMMs as a
     cache for the AEP DIMMs.
  2. Filesystems, in which you first enable 1LM mode, then install a filesystem
     onto the AEP DIMMs to take advantage of their persistence.
  3. NUMA mode, in which you enable 1LM mode, then bind the AEP DIMMs' memory regions
     to a driver which makes them available as a NUMA node.

For my research, I'm particularly interested in the third option, since most of
my tools assume that memory tiers will be accessible as NUMA nodes, and
onlining a set of AEP DIMMs as a NUMA node would allow me to use those tools
without modification.

## Preparation

In general, these are the steps to prepare to go into NUMA mode:

  1. Ensure you've got a new enough kernel. It needs to be at least version `5.1.0-rc4`.
  2. Change the AEP to `1LM` mode in the BIOS.
  3. Use `ipmctl` to put each individual AEP DIMM into AppDirect mode:
     To do this to all AEP on socket 0, do:

     `ipmctl create -goal -socket 0 PersistentMemoryType=AppDirect`
     Reboot.

  4. Now the DIMMs that you chose from the previous command will show as one distinct memory region in
     `ipmctl`. However, you need `ndctl` and `daxctl` to bind this region to the appropriate kernel driver.

     `git clone https://github.com/pmem/ndctl.git`

  5. Build the project.
  6. Migrate the device model. This writes to the file `/etc/modprobe.d/daxctl.conf`. If it doesn't, then
     you didn't globally install `daxctl` to your system (it hardcodes some paths, so running this command from
     the source directory doesn't work). Reboot again.
  7. View the memory regions that you created with `ipmctl`, using `ndctl`:
     
     `ndctl list -R`

  8. Create a namespace consisting of that region:

     `ndctl create-namespace --region region0 -m dax`

  9. Ensure you *don't* have `dax_pmem_compat` running: 

     `lsmod | grep "dax_pmem_compat"`

  10. Ensure you *do* have `kmem` running:
     
      `lsmod | grep kmem`

## The Manual Way

The next step is where I've been confused. Initially, my version of the
`daxctl` command didn't include any other commands: that is, from here on out,
you had to manually bind the namespace to the proper kernel driver. To do this
manually (again for a single memory region):

  1. Find the device that you want to bind. Examples include `dax0.0`,
     `dax1.0`, etc. This will list the namespaces, and within these should be
     the device that they're associated with:

     `ndctl list -N`

  2. Unbind from the `device_dax` driver:

     `echo dax0.0 > /sys/bus/dax/drivers/device_dax/unbind`

  3. Bind to the `kmem` driver:
   
     `echo dax0.0 > /sys/bus/dax/drivers/kmem/new_id`

Once that's done, you should now check `numastat -m` to see if you've got a
newly-created NUMA node. If it contains the capacity that you expect it to be,
you're done.

## The Automatic Way

However, one time I've encountered a situation in which this wasn't the case.
Upon checking `numastat`, the node had a size of 0 bytes. Internet searches
didn't seem to give any hints as to why this was. As it turns out, getting the
newest version of `daxctl` (which includes the `online-memory`,
`offline-memory`, and `reconfigure-device` commands) fixed my issue, but with a
caveat.

Using a new enough `daxctl`, the last three steps can be replaced by a simpler:

`daxctl reconfigure-device --mode=system-ram all`

This will unbind the device from the old driver and rebind it to the new
driver.  Crucially, though, it also *onlines* the memory regions, which is what
I was missing when I encountered the 0-size NUMA node issue. Using this new
method, the node now shows up with the appropriate capacity.

## More Problems

On my system, I have two NUMA nodes of DDR, 0 and 1. Each of these is 96GB.
Node 2 is the AEP on socket 0, while node 3 is the AEP on socket 1. So-- to
bind an application to the memory on socket 1 (while preferring DDR and spilling
onto AEP), I do:

`numactl --preferred=1 numactl --membind=1,3 --cpunodebind=1 ./a.out`

For applications that use less than approximately 200GB of RAM, this works just
fine.  However, upon scaling them to use more than 200GB, I start encountering
issues: the OOM killer is suddenly killing my process, and if I set
`vm.overcommit_memory` to `2` (thus forcing `malloc` to return `NULL` before
allocating more memory than is available), the applications fail to allocate
memory above ~200GB.

Searching for this issue doesn't return many results, either, and I can't seem
to figure out why those allocations are failing. If I ignore the DDR node,
binding only to the AEP, all allocations succeed, and I can scale my
application to use a peak RSS of more than 700GB. However, immediately upon
preferring the DDR and spilling onto one of the AEP nodes, the kernel OOMs upon
reaching around 200GB, despite there being nearly 600GB of free memory available
on node 3.

## Memory Zones

Nearly giving up, I finally check each of the memory regions that make up the
AEP NUMA nodes. For node 3, checking the first gigabyte of memory looks like:

`cat /sys/bus/node/devices/node3/memory1000/state`

The value was `online`. However, upon checking `valid_zones` in the same
directory, it seems that the memory is in `ZONE_MOVABLE`, not `ZONE_NORMAL`.

[Reading up on this](https://lwn.net/Articles/224255/), it explains why I could
manually bind to specifically that node and use its fully capacity, but not
fault memory onto it. Since this memory was onlined as `ZONE_MOVABLE`, the most
that the kernel can access is the minimum of what the other nodes have: that
is, the more-than-700GB node of AEP can only have ~96GB faulted onto it, so
that I get an OOM when I allocate above nearly 200GB of memory (96GB of DDR,
plus 96GB of AEP which I fault to). This also explains why binding directly to
that node succeeds: userspace applications can still use the full capacity of
the node just fine, and `numastat -m` shows the full amount.

Searching further, I find out why the nodes were `ZONE_MOVABLE`: the `daxctl` command
onlines them that way, by doing e.g.

`echo online_movable > /sys/bus/node/devices/node3/memory1000/state`

to each of the `memoryXXXX` directories for a particular node. While this is
fine for an application that binds directly to that node, a subtle issue is
that the node cannot be fully faulted onto if it has more capacity than the
minimum of all of your other NUMA nodes (as will usually be the case for AEP).

## The Solutions

The first and simplest solution is to modify `daxctl`. I chose this one, as it
was the quickest to implement. For release version `v66`, I edited line 1095 of
`daxctl/lib/libdaxctl.c` from `online_movable` to `online`, then recompiled,
rebooted, and re-onlined the memory with this new version.

For those that want to use the manual method, there are two more solutions.
First, you can simply write a script that manually onlines each of the
gigabytes of memory for your AEP regions. Your script would essentially iterate
over each of the directories in `/sys/bus/node/devices/nodeX/`, and `echo
online > state` for each of the `memoryXXXX` directories.  This would most
likely be the simplest solution if you try the manual method above, but end up
with a 0-size NUMA node (and don't want to use `daxctl`).

The solution that would be easier in the long-term, though, is to enable the
kernel configuration `CONFIG_MEMORY_HOTPLUG_DEFAULT_ONLINE`, which
automatically onlines newly-hotplugged memory. The memory would then be
immediately onlined into `ZONE_NORMAL` upon being bound to the `kmem` driver.
