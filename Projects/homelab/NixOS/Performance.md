# Performance

A Linux machine can be tuned for better performance depending on your workflow. Kubernetes clusters with lots of components running tend to be quite RAM-hungry, and tuning your machine such that `zramSwap` is enabled, for example, can help in dealing with RAM-heavy moments or workloads. Let's have a look at `performance.nix` to see what was tuned for my specific use cases and why:

`modules/performance.nix`
![[Pasted image 20261013204410.png]]

This module holds many configurations, some of which are overridable per node if the need arises in the future. Let's have a look at the first:

```nix
  boot.kernelPackages = lib.mkDefault pkgs.linuxPackages_latest;
```

The first line is telling Nix that the Linux kernel version that's going to be used is, by default, the latest one. `lib.mkDefault` is responsible for setting the value of `boot.kernelPackages` if and only if the value is not present already. There is a kind of ordering when declaring what value a certain config will get, and it goes as follows from lowest priority to highest:

```nix
# Lowest
x = lib.mkDefault y

# Middle
x = y

# Highest
x = lib.mkForce y
```

Setting the kernel version as a default value is what allows us to override this configuration depending on the node and what our wishes are. So far, both of my nodes hold the same version, but this might not be what I want in the future. Hence, I have taken preemptive measures.

Next, we have some kernel parameters:
```nix
  boot.kernelParams = [
    "transparent_hugepage=madvise"
    "psi=1"
    "mitigations=off"
  ];
```

`transparent_hugepage=madvise` lets processes that may benefit from [THP (transparent huge pages)](https://lwn.net/Articles/359158/) use it, while defaulting to the normal page size for other processes that don't explicitly call it. THP is a feature of the Linux kernel which allows for larger OS pages. OS pages in x86_64 Linux are 4kb, but with THP they can be 2mb. THP is only really good for a couple of things and can be horrible for many others. Linux gives each process its own virtual memory address space. Whenever a process asks for memory, it's given a range of virtual addresses to [memory pages](https://en.wikipedia.org/wiki/Page_(computer_memory)), and for a page to be used, its virtual address needs to be translated into a physical address. To avoid doing this address translation for frequently accessed addresses, a small cache of virtual-to-physical addresses called the TLB (translation lookaside buffer) is used. Say you allocated 2gb of memory. Normally, this memory buffer will be divided into 4kb memory pages, which amounts to ~500k pages in total, which cannot all fit into the TLB (~2k entries in total). Randomly accessing parts of this memory buffer (such as in the case of a hash map) would result in continuous TLB misses and memory translation costs. Now let's say THP was enabled for our process. This would make our 2gb slab of memory fit into just 1024 memory pages, all of which fit in a typical x86 TLB, speeding up our workflows. Virtual-to-physical address translation in Linux works through a 4-level radix tree, where each node in the tree has 512 children. A virtual address is made up of 48 bits. Translation works by traversing this tree using the bits of the virtual address as the path. Some of the bits within the virtual address are used to find the location of the memory page, while the others are used to locate the exact memory address inside the page. Since pages are typically 4kb in size, we need log2(4kb) bits to store an address within the page, amounting to 12 of the bits being used just to access the page after finding it. Now, we're left with 36 bits, which are then further divided into 4 sections of 9 bits each. Each section is used on each level to go further down the tree, with the 9 bits representing the number of the child to go to for each level. So, doing address translation for 4kb pages requires a traversal of 4 levels. For a 2mb page, this is different: log2(2mb) results in 21 bits instead of 12, leaving 27 bits needed for tree traversal. Splitting these 27 bits into sections of 9 leaves us with 3 levels of the tree needed instead of 4, which lets us skip one level of the tree for faster overall translation. This last part is a very, very slight optimization, but it's worth mentioning since it's a cool part of the inner workings of CPUs and OSes.

Transparent huge pages can cause a lot of issues if misused. Whenever a process demands a page of memory, the OS has to allocate one from RAM. Allocating a 4kb memory page is significantly easier than allocating a 2mb one. As a matter of fact, allocating 512 4kb pages is easier than allocating a single 2mb page due to [memory fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)). Continuous use and misuse of huge pages can lead to periods of massive stalls where the OS is struggling to find a contiguous 2mb slab of memory, significantly slowing down your processes rather than speeding them up. Other issues occur in processes that use `fork()` and write to memory originally allocated in the parent process. Whenever a `fork()` is used, memory pages from the parent are kept as read-only for both processes. `fork()` itself doesn't do any copying. The moment a child or parent process writes to any of those pages, no matter how small the write is, and more than one process still has access to that page, the page is fully copied and handed to the process to do as it pleases. For 4kb, that copy is not a huge deal, whereas a 2mb huge page is a huge deal and may slow things down or consume too much RAM quite dramatically.

`psi=1` enables "Pressure Stall Information", which lets tools like the node-exporter scrape information about which processes are stalled waiting for resources and by how much. This is useful when debugging why a service might be slow on the lower-level side of things.

`mitigations=off` disables a security feature of CPUs meant to protect them from speculative execution attacks. CPU performance can be a real bottleneck in my `tux` node. It is slightly less of a bottleneck in my `cachyos` node, but I've found this helpful nonetheless. The security impact of this is minimal in my case since the only workloads I'm running are the ones I create.

Now for some RAM and swap tuning:
```nix
  zramSwap = {
    enable = true;
    algorithm = "zstd";
    memoryPercent = 50;
  };
```

Using swap memory is what operating systems tend to do when they run out of RAM. If the demand for RAM doesn't meet supply, your OS offloads some of the RAM that isn't being actively used to a disk partition specifically for swap. The moment that data is needed again, the OS loads it back into RAM, offloading some other part to swap if there still isn't enough space. `zramSwap` is similar, but instead of offloading to disk, your OS compresses some memory pages and holds that compressed data in RAM, decompressing it when that data is demanded again. These methods of swap aren't mutually exclusive, and in fact, Windows 11 uses both by default. In my case, I have opted to use just `zramSwap` instead of utilizing the disk for it. Both methods have their pros and cons. Traditional swap memory can hold a lot of data. Disk memory is less expensive and bigger than volatile memory, but its speed is limited by your hard disk. Even SSDs can be slow when there's lots of I/O demand from other processes. `zramSwap`, on the other hand, trades some capacity for speed. The CPU is quite fast at compression and decompression, significantly faster than your disk is at reading and writing, but compression can only save so much space. It can be especially ineffective at compressing seemingly random data as in the case of encrypted data. Nonetheless, the swap mechanism is a necessity in any workload where sudden, short-lived spikes in RAM demand that exceed your supply tend to occur.

The `zstd` algorithm tends to be the slower option, but it saves the most data. Other algorithms like `lz4` and `lzo` are faster but compress worse. `memoryPercent` defines the maximum amount of RAM `zramSwap` is allowed to use for holding compressed data within RAM itself. As I've come to find out, most people recommend 50% as a good default value.

Now for some tuning of kernel runtime parameters:
```nix
  boot.kernel.sysctl = {
    "vm.swappiness" = 100; # zram-appropriate
    "vm.overcommit_memory" = 1;
    "vm.max_map_count" = 1048576;
    "fs.inotify.max_user_instances" = 8192;
    "fs.inotify.max_user_watches" = 524288;
    "fs.file-max" = 2097152;
    "net.ipv4.ip_forward" = 1;
    "net.bridge.bridge-nf-call-iptables" = 1;
    "net.core.somaxconn" = 4096;
    "net.netfilter.nf_conntrack_max" = 262144;
    "vm.dirty_background_ratio" = 5;
    "vm.dirty_ratio" = 10;
    "fs.aio-max-nr" = 1048576;
    "net.ipv4.tcp_congestion_control" = "bbr";
    "net.core.default_qdisc" = "fq";
  };
  
  boot.kernelModules = [ "tcp_bbr" ];

```

`vm.swappiness` is well above the default value of `60`. This number refers to how likely the kernel is to use swap memory as opposed to loading a page back to RAM from swap. In most cases where swap uses the disk, you want this value to be at its default or lower, since frequently offloading RAM can be slow. But here we are using `zramSwap`, which can load and offload memory at significantly higher speeds than the disk can, making swap memory usage something we want to actively encourage for better memory management.

`vm.overcommit_memory` lets processes allocate more memory than is available on the machine. Most arena allocators tend to double their capacities over time, which can result in some huge memory demands that can be very hard to meet while barely utilizing all of the demanded memory anyway. Arena memory allocators are common in high-performance applications such as databases or something like the JVM. Overcommitting memory is possible due to virtual memory. Whenever a process initially asks for memory, the OS just marks that memory as allocated virtually. Actual RAM is only ever allocated when demand is present. The OS lazily maps virtual pages to physical ones during a process's lifetime. So if your machine has 8GB of RAM, and a process asks for a 10GB slab of memory, the OS only allocates RAM for the parts of that 10GB of virtual memory the process actually uses. `fork()` can also cause this due to duplicating the virtual address space of the parent, even though the child might not use all of it.

`vm.max_map_count` controls how many `mmap()` regions are allowed for a single process. Current demand does not justify an increase from the default level, but my future plans will probably match this.

`vm.dirty_background_ratio` and `vm.dirty_ratio`: these two configs are similar and work together. Whenever something is written to a file, the OS doesn't outright flush that write to disk. Instead, it keeps that write in memory pages and flushes them when appropriate. `vm.dirty_background_ratio` sets the percentage of RAM the OS can use before flushing pages to disk gradually and asynchronously. `vm.dirty_ratio` is similar, but instead of writes having to be async, they become blocking. No more RAM is allowed to be used for file writes than what this config is set to, and it will block future file writes until the existing pages are flushed out. For critical workloads where data loss cannot be tolerated, keeping this number low can be beneficial (for example, WALs which don't fsync right away may lose more data if the number is too big). 

`fs.inotify.max_user_instances`: inotify is used by processes that watch files for changes. Kubernetes uses this a lot, and the default value for this parameter tends to be too low.

`fs.file-max` controls how many file descriptors are allowed to exist at once. My nodes currently run lots of pods and will only run more pods in the future, causing an increase in file descriptors. It's better to make some headroom now than to have to debug pods crashing because of something I'd never think to check.

`fs.aio-max-nr` refers to the maximum total number of async I/O requests that processes are allowed to have on this machine. There really is no limit to how high the number can be, and setting the limit to a high number can help processes that make a lot of async I/O requests, such as databases. The only reason to set a number that is not massive is to prevent misbehaving processes from exhausting all kernel memory by making unlimited I/O requests.

`net.ipv4.ip_forward`, when enabled, allows machines to route packets between interfaces. This is a requirement for pod-to-pod communication in Kubernetes.

`net.bridge.bridge-nf-call-iptables` also needs to be enabled for Kubernetes, specifically for `NetworkPolicies`. Pod network traffic is bridged traffic, and this setting lets `iptables` control bridged traffic, which `NetworkPolicies` can control. 

`net.core.somaxconn` impacts how TCP sockets behave. Whenever an incoming TCP connection arrives at a socket, it sits in a queue until that connection is `accept()`-ed. This config controls how big that queue can be for each TCP socket. Setting it to a high number can help prevent dropped connections under load. Note that this is just a ceiling. Processes can still choose to have smaller queues.

`net.netfilter.nf_conntrack_max` sets the maximum number of network connections the kernel can keep track of. [Netfilter](https://www.netfilter.org/) is what the kernel uses to filter packets as well as hold information about network connections. Data about the connections includes the source/destination IP addresses and ports of packets, as well as NAT-related data such as what the packet's source/destination was rewritten to or from. Kubernetes specifically puts this connection tracking under a lot of load due to how `kube-proxy` manages the `Service` resource. The value that was set for this configuration allows for a lot of headroom. Errors from a misconfigured number can be a pain to debug.

`net.ipv4.tcp_congestion_control`, `net.core.default_qdisc`, and `boot.kernelModules = [ "tcp_bbr" ]` are configs that work together. `boot.kernelModules = [ "tcp_bbr" ]` makes BBR available to the Linux kernel as a congestion control algorithm that can be selected in other configs. A congestion control algorithm decides how to send packets as fast as possible without having them dropped by your router. `net.ipv4.tcp_congestion_control = "bbr"` uses the algorithm that was loaded by the previous setting. `net.core.default_qdisc` is a configuration for the layer sitting between the kernel's TCP/IP stack and the network interface driver. It's usually the case that more than one socket is ready to transmit packets onto the wire, but only some do at a time. `qdisc` decides which ones, and in my case the algorithm behind it is `fq` (fair queue). The default congestion control algorithm for Linux is `cubic`, which works by continuously sending packets faster and faster until your packets start to get dropped. Once the drops happen, it sees that as the signal to cap its rate. This algorithm rests on the assumption that packets you send are only ever dropped due to congestion. In cases such as nodes being connected over Wi-Fi, this is not always the case, and the algorithm may cap your speeds for no reason other than an unexpected event that dropped packets. `bbr`, on the other hand, doesn't measure through loss; instead, it relies on the bandwidth and RTT of the network and uses those factors for congestion control. As for `qdisc`, the default algorithm for Linux is just FIFO: the first packets to be queued are the first to be sent. `fq`, on the other hand, has a separate queue per network flow and the ability to send packets at a specific time. `bbr` needs these capabilities in order to properly space out packets according to its calculations.


Next, we have:
```nix
services.fstrim.enable = true;
```

SSDs can only write to blocks of their memory that are marked as free. Whenever you delete a file in Linux, the only guarantee present is that Linux will mark that file as deleted. Your SSD doesn't know if the block corresponding to that file is free yet. The OS has to tell it that. Telling your SSD that a block is free is an operation that takes some non-zero amount of time. As such, doing it the moment you delete a file might slow your processes down (the call to do it is synchronous). The next best option is to periodically inform the SSD of free blocks in batches, during periods when the SSD is already idle, rather than making processes wait to confirm that a file is really deleted.

Disk scheduling:
```nix
  services.udev.extraRules = ''
    ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
    ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
  '';
```

`udev` lets you add extra rules for custom handling of device-related events. In this case, two extra rules were added:

1. Whenever a device whose name matches the `nvme[0-9]n[0-9]` regex is added or changes, set its `queue/scheduler` attribute to `none`.
2. Whenever a device whose name matches the `sd[a-z]` regex is added or changes, and its `queue/rotational` attribute is `0` (meaning it's not a spinning disk), set its `queue/scheduler` attribute to `mq-deadline`.

The OS has I/O scheduling abilities: whenever some writes are queued, it can decide the order in which those writes are carried out. In the case of spinning disks, this was highly beneficial, as reducing the seek time between each write meant saving a considerable amount of time. SSDs don't have these seek times because they don't have a moving head, so the benefits aren't as great, but it's still a worthy inclusion for SATA drives. SATA drives (like in the case of `tux`) have one queue for processing reads and writes. A machine running different processes in parallel, each of which will have its own read and write requests, would need to coordinate these requests so that one process doesn't starve every other process by hogging the entire queue. NVMe drives, on the other hand, have multiple queues, which reduces sharing of a given queue between different CPUs, which in turn requires less coordination and therefore less work. 

Configuring `journald`:
```nix
  services.journald.extraConfig = ''
    SystemMaxUse=500M
    RateLimitIntervalSec=30s
    RateLimitBurst=10000
  '';
```

I'm not a big fan of `journald`. `journald` is what `systemd` uses for storing and retrieving logs it got from its units. `journald` stores those logs in a binary format, which is efficient for storage but not human-readable. If your `journald` crashes or gets corrupted for whatever reason, retrieving those logs can become a real hassle. Nonetheless, NixOS and `systemd` (therefore `journald` as well) are very tightly coupled with each other, so I have no choice in this area. `SystemMaxUse=500M` puts a limit on how large the journal can grow from storing logs. `journald` computes its own limit by default, but it may be larger than you want. `RateLimitIntervalSec=30s` and `RateLimitBurst=10000` cap how fast a unit is allowed to print out logs. If a process prints more than `10000` logs in `30s`, `journald` starts dropping future logs and keeps count of those dropped logs. Any process continuously crashing and restarting will therefore not consume all of your log space. 

And finally:
```nix
  systemd.oomd.enable = false;
```

`systemd.oomd` is very useful. It uses PSI stats to determine when to OOM-kill (out of memory) processes, as opposed to waiting until the kernel is *completely out of memory* before acting. However, Kubernetes already does this on its own using `kubelet`. `kubelet` and `systemd.oomd` can't communicate and cooperate with each other, which can lead to more processes being OOM-killed than necessary. Since turning off `kubelet` would lead to this not being a Kubernetes cluster in the first place, I just disabled `systemd.oomd`.

---

← Previous: [[Projects/homelab/NixOS/Common Configuration|Common Configuration]] | Next: [[Projects/homelab/NixOS/Laptop Configuration|Laptop Configuration]] →

[[Projects/homelab/Nixos|NixOS Overview]]
