---
layout:     post
title:      Memory Mode
date:       2019-10-17
summary:    How to use AEP DIMMs in Memory Mode.
categories: jekyll
---

An earlier post describes how to get into NUMA mode, but I figured I'd write down how to get back into
2LM (or Memory Mode, or Cache Mode) for future reference.

1. Switch to "2LM" mode in the BIOS.
2. Disable and destroy all namespaces:
  {% highlight BASH %}
  ndctl list -N
  ndctl disable-namespace namespace0.0
  ndctl destroy-namespace namespace0.0{% endhighlight %}
3. Disable all regions:
  {% highlight BASH %}
  ndctl list -R
  ndctl disable-region region0
  ndctl disable-region region1{% endhighlight %}
4. Put all of each socket's memory in Memory Mode:
  {% highlight BASH %}
  ipmctl create -goal -socket 0 MemoryMode=100
  ipmctl create -goal -socket 1 MemoryMode=100{% endhighlight %}
5. Reboot again.
