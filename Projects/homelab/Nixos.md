>[ Nix is a tool that takes a unique approach to package management and system configuration. Learn how to make reproducible, declarative and reliable systems.](https://nixos.org/#:~:text=Nix%20is%20a%20tool%20that%20takes%20a%20unique%20approach%20to%20package%20management%20and%20system%20configuration%2E%20Learn%20how%20to%20make%20reproducible%2C%20declarative%20and%20reliable%20systems%2E)

NixOS is an operating system built on top of the Nix package manager. It lets me declare how I want my nodes to be, down to the finest details, and then makes that declaration a reality. Many of its features make it an attractive tool for managing the configuration of your servers, such as the following:

- Reliable Upgrades - Nix is a purely functional package manager, meaning that the same parameters will always yield the same result. It's particularly useful for repeatable builds.
- Atomic Upgrades - NixOS takes a transactional approach to configuration upgrades. If you want to change the config of one of your servers and, after applying the changes, the upgrade process fails or is interrupted for whatever reason, the entire upgrade attempt is aborted, and your server is in the exact same state as it was prior to the config upgrade.
- Rollbacks - Each time `sudo nixos-rebuild switch` is used to upgrade your configuration, its old versions are still stored on your machine. You have the ability to switch to any of those old versions. Even if you were to delete those old versions through `nix-collect-garbage -d`, the power of Git in this repository allows me to revert any commit such that my config is back to an older version, which I can then just apply.

The power of NixOS is not limited to what is mentioned here and surpasses my human understanding, but the above section should provide an idea as to why anyone would choose to use this tool.

One particularly useful feature of NixOS is [Nix Flakes](https://nixos.wiki/wiki/flakes). According to the docs:
>[Nix flakes provide a standard way to write Nix expressions (and therefore packages) whose dependencies are version-pinned in a lock file, improving reproducibility of Nix installations.](https://nixos.wiki/wiki/flakes#:~:text=Nix%20flakes%20provide%20a%20standard%20way%20to%20write%20Nix%20expressions%20%28and%20therefore%20packages%29%20whose%20dependencies%20are%20version%2Dpinned%20in%20a%20lock%20file%2C%20improving%20reproducibility%20of%20Nix%20installations%2E)
<img style="display: block; margin-left: auto; margin-right: auto; width: 50%;" src="Pasted image 20261013204243.png"  alt="filetree" />
# The Entry Point

Everything begins with `flake.nix`. Usual function definitions go something along these lines: input -> logic -> output. The contents of the `flake.nix` file are no different:


![[Pasted image 20260826224348.png]]

>[!NOTE]
>The `nixpkgs/nixos-25.11` version has been updated to pull in the newest Linux kernel.

A Nix flake consists of `inputs`, `outputs`, and a `description`. The description makes no difference in functionality. In `inputs`, you declare what your flake needs in order to work, and in `outputs`, you declare what your flake will create based on those inputs. Within this flake, I have declared that my inputs will be the Nix packages repository, `disko` (a third-party module that allows for declarative disk layouts and partitioning), and `sops-nix` (a third-party module for managing secrets in my repo). And in my outputs, I have first declared a helper function in Nix called `mkNode`, which takes as an argument the `hostname` of a node and returns a `nixosSystem` configuration for that specific `hostname`. Then I have declared that my `nixosConfigurations` will be an attribute set (think dictionaries in Python: just a set of key-value pairs) where the values are nodes created from hostnames. As a result, every node consists of the same modules, differing only in those modules that depend on the hostname itself. This approach provides code reusability and minimizes duplication while keeping shared behavior between nodes consistent. It will be especially useful once I have more nodes, since adding a node is as trivial as adding a couple of files and modifying this attribute set.
# Shared modules

The entry point creates `nixosSystems` from shared modules as well as node-specific modules. Since shared modules are present on all nodes, I'll be covering them first.

## Commons

`modules/common.nix`
![[Pasted image 20260827223508.png]]

Every module gets a list of arguments available to it from the flake. The list at the top is not exhaustive, meaning any given module may have access to more arguments than are declared. It's just that the only declared arguments are the relevant ones for this module. `config` is an argument that contains contributions from every module and makes those contributions available to all modules that use it. Nix doesn't evaluate modules in file order. It instead first combines every contribution to the `config` object from every module and then starts evaluation. In our case, `common.nix` is using a value from config.sops.secrets, which contains the path to the hashed password of the `slowking` (me!) user. Secrets management for the NixOS part of the project will be explained in detail later on. For now, just keep in mind that `config.sops.secrets.*` provides us with different values which are safely encrypted and stored in this repository. `pkgs` contains the packages repository provided by the Nix package manager, and `lib` provides utility functions for manipulating objects in Nix.

After the list of arguments, you'll find I have declared a helper list, which contains the SSH public keys of different machines I use to SSH into my nodes. I later reference this list in other configurations below. Since they are public keys, it's completely safe to have them out in the wild like this!

Below the list of arguments, different configurations for a node are provided. I'll walk through each one.

```nix
time.timeZone = "Europe/Athens";
i18n.defaultLocale = "en_US.UTF-8";
```
The above code snippet serves to set the time zone and language of the system.

```nix
  nix.settings = {
    experimental-features = [ "nix-command" "flakes" ];
    auto-optimise-store = true;
    trusted-users = [ "root" "deploy" ];
  };
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 14d";
  };
```
This snippet contains configs for Nix itself. The first attribute set declares that Nix may use the described experimental features, can use `auto-optimise-store`, and has 2 trusted users: `root` and `deploy`. The deploy user is a special user who may use passwordless sudo in order to run `nixos-rebuild switch` to apply new configurations. This part will be explained in more detail later. Below that, you'll find configurations for the Nix garbage collector, a crucial component of Nix which ensures the cleanup of packages that are no longer used by your system. In this code snippet, I'm declaring that garbage collection runs automatically every week and deletes packages that are no longer used AND are older than 14 days. The 14-day condition is there to ensure that packages from older generations of configs can still be present if I decide to roll back.

```nix
users.mutableUsers = false;
```

This line forbids editing users and groups by hand. The one and only way to edit users and their configurations is through nix. By keeping users immutable, I prevent someone (like a dumber me in the future) from editing users through SSH and then forgetting about it (the main issue NixOS solves in the first place).

```nix
  users.users.slowking = {
    isNormalUser = true;
    uid = 1000;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = adminKeys;
    # Interactive password sudo for hands-on admin
    hashedPasswordFile = config.sops.secrets."slowking/hashed-password".path;
  };

  users.users.deploy = {
    isNormalUser = true;
    uid = 1001;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [ (lib.strings.trim (builtins.readFile ../keys/deploy.pub)) ];
  };
```

Here I have configured two users who are present on all nodes. The first user is `slowking` (me!), and the second is the `deploy` user. For the `slowking` user, I have specified that they are a normal user, which just means that logging in and having a shell is possible for this user. They have UID 1000, which can stay unspecified. However, if it does stay unspecified, then NixOS will auto-assign a UID to that user, and if I were to add users in the future, then my slowking user might get a new random auto-assigned UID while there are still files in the system created by slowking using the old UID. Pinning it manually resolves this issue. They are part of the `wheel` group, which is a necessity for using the `sudo` command. You can log in as the slowking user via SSH with any of the admin keys listed by the `adminKeys` list, and finally, their password hash exists in a file whose path is specified in sops secrets. 

The `deploy` user has much the same characteristics, except that only one public SSH key is allowed to log in as this user and they don't have a password (meaning that the only way to log in as this user is via SSH).

```nix
  # deploy may run nixos-rebuild's activation non-interactively.
  security.sudo.extraRules = [{
    users = [ "deploy" ];
    commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
  }];
```

Above I have defined some extra security rules. Specifically, I have declared that the `deploy` user may use all commands and may use sudo without a password. This isn't true for the `slowking` user, who still needs a password to use sudo.

```nix
  environment.systemPackages = with pkgs; [
    git vim htop btop tmux iproute2 iptables ethtool pciutils usbutils
    dnsutils curl jq fastfetch aha
  ];
```

And lastly, for the `common.nix` file, I have specified what system packages are available on my nodes. NixOS looks at the list of specified packages within `environment.systemPackages` and ensures your system will have them downloaded and ready to be used after applying the changes. The `with pkgs;` bit lets us omit the `pkgs.*` part before each entry in the list and makes for a cleaner-looking list. The name of each entry correlates to the name of a package within the Nix package manager.

## Performance

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

## Laptop Specifics

Laptops have functionality that is not present in most machines and can actually hinder a node being used as a server. However, there are remedial steps we can take to make up for this difference. In particular, laptops want to prioritize energy efficiency, and they achieve this by throttling your CPU's performance or suspending your machine under certain conditions. Both work against a server's interests, especially the latter. Therefore, I created a Nix module containing laptop-specific configs that nodes can then optionally import to prevent these issues:

`modules/laptop.nix` 
![[Pasted image 20261013205004.png]]

This module consists of two blocks of configs; I will walk through them:

```nix
  services.logind.settings.Login = {
    HandleLidSwitch = "ignore";
    HandleLidSwitchExternalPower = "ignore";
    HandleLidSwitchDocked = "ignore";
  };
```

The block above prevents `logind` from taking action when the mentioned events occur. All of these events (closing the lid, pressing the power button once, etc.) would cause a laptop to suspend its action, a most horrible situation for a server that's supposed to be on as much as possible.

Next, we have some power and performance settings:
```nix
  powerManagement.enable = true;
  powerManagement.cpuFreqGovernor = "performance";
  services.power-profiles-daemon.enable = lib.mkForce false;
  services.tlp.enable = lib.mkForce false;
```

All of these features do more or less the same thing. My nodes are laptops and, as such, they don't require max throughput and power all day every day. Most OSes running on laptops tend to have settings enabled which limit the performance of the laptop in order to save some battery life. My nodes have been repurposed as servers and are plugged in at all times. Therefore, these settings are only a hindrance for my use case.

## Networking

The networking section for NixOS primarily deals with the `networking.nix` file. There, you'll find configurations regarding the network setup of nodes to ensure stable communication with each other and the internet. The primary use of this module is for reusability across nodes, and it is to be imported into the custom definitions of a node.

`modules/networking.nix`
![[Pasted image 20260908194039.png]]

At the top of the file, I've defined a helper variable to be reused below, which carries the value of `config.homelab.node`. Instead of typing `config.homelab.node` every time I need a value from it, I can just say `cfg`.

```nix
  options.homelab.node = {
    wifiInterface = lib.mkOption { type = lib.types.str; description = "WiFi interface name"; };
    ipv4 = lib.mkOption { type = lib.types.str; description = "Static IPv4 address (no prefix)"; };
  };
```
The above snippet creates attributes that can later be referenced in other modules and in this module. Here I have simply declared that they exist, what their type will be, and a description of what they hold for clarity.

`homelab.node` now holds a `wifiInterface` option, which is then set by other modules and can be node-specific if needed. It also holds `ipv4`, which, similarly to the previous one, can be referenced by other modules for their uses.

```nix
config.networking = {
  useDHCP = false;
  enableIPv6 = false;
  wireless = {
    enable = true;
    interfaces = [ cfg.wifiInterface ];
    secretsFile = config.sops.secrets."wifi/env".path;
    networks.${lib.strings.trim (builtins.readFile ../assets/wifi-ssid.txt)}.pskRaw =
      "ext:WIFI_PSK";
  };
  interfaces.${cfg.wifiInterface}.ipv4.addresses = [
    { address = cfg.ipv4; prefixLength = 24; }
  ];
  defaultGateway = "192.168.1.1";
  nameservers = [ "192.168.1.1" ];
  extraHosts = ''
    192.168.1.25 registry.gentoo.lan
  '';
};
```

This snippet sets some of the values within the `config` attribute set that's present in every module. There is no difference between saying
```nix
config.networking.useDHCP = false;
config.networking.enableIPv6 = false;
```
and
```nix
config.networking = {
  useDHCP = false;
  enableIPv6 = false;
};
```

But I think we can agree one is nicer to look at.
`useDHCP = false` disables the use of DHCP. These nodes need stable IPs in order for the Kubernetes control plane to know how to find its workers and for the workers to know how to find the API server.
`enableIPv6 = false` disables the use of IPv6. My current Wi-Fi network has no IPv6 capabilities, which would cause connections via IPv6 addresses from my pods to hang indefinitely. I ran into this issue when my pods were failing to pull some images from [docker.io](https://www.docker.com/). Current networking library implementations (glibc) prefer the use of IPv6 addresses over IPv4, and when a DNS query returned an IPv6 address, that's what caused my pod to hang.
```nix
    wireless = {
      enable = true;
      interfaces = [ cfg.wifiInterface ];
      secretsFile = config.sops.secrets."wifi/env".path;
      networks.${lib.strings.trim (builtins.readFile ../assets/wifi-ssid.txt)}.pskRaw =
        "ext:WIFI_PSK";
    };
```
This block enables wireless connectivity and sets the network interface for wireless to be the value we set at `cfg.wifiInterface`. The `secretsFile` attribute wants the path to the password key for the Wi-Fi we are connecting to, a value that is provided by a secret we manage via `sops`. The following line is a bit more complicated, but it's simple once you understand what is going on:

`networks.${lib.strings.trim (builtins.readFile ../assets/wifi-ssid.txt)}.pskRaw = "ext:WIFI_PSK";`

First, let's look at this piece of Nix code: `${lib.strings.trim (builtins.readFile ../assets/wifi-ssid.txt)}`. Anything inside a `${}` bracket is evaluated, and its value then replaces the whole expression at evaluation time. So if that expression were to evaluate to `slowking`, what Nix would then see in my config is
`networks.slowking.pskRaw = "ext:WIFI_PSK"`. We can understand the code inside the `${}` bracket as follows: `lib.strings.trim <string>`, which just means we are calling the function `lib.strings.trim`, which takes a string as an argument and removes its trailing newline. The value of `<string>` is determined by `(builtins.readFile ../assets/wifi-ssid.txt)`, and all that bit does is call the `builtins.readFile` function, which takes a path to a file as its argument and returns the contents of that file as a string. So, all in all, that entire `${...}` section just evaluates to the trimmed contents of a file, and what's in that file is the name of my Wi-Fi network! Further on that line, we see `networks.${name}.pskRaw = "ext:WIFI_PSK"`, which works with the previous line we saw with the `secretsFile`. `.pskRaw` may hold the raw passkey to the wireless network, but `"ext:WIFI_PSK"` tells the machine to use the value present in the `secretsFile` we defined before.

```nix
    networking.interfaces.${cfg.wifiInterface}.ipv4.addresses = [
      { address = cfg.ipv4; prefixLength = 24; }
    ];
```

This snippet uses the option we defined at the beginning, namely `cfg.ipv4`. Here we are defining the addresses for the network interface `${cfg.wifiInterface}` as `[ { address = cfg.ipv4; prefixLength = 24; } ]`. So there is just one address, whose value is taken from `cfg.ipv4`, which is a value set by each node itself (more on that later on).

```nix
    networking.defaultGateway = "192.168.1.1";
    networking.nameservers = [ "192.168.1.1" ];
```
It's simple enough: the machine will use the address `"192.168.1.1"` as its default gateway and as its DNS server.

```nix
    networking.extraHosts = ''
      192.168.1.25 registry.gentoo.lan
    '';
```
If you've ever edited the `/etc/hosts` file on an Ubuntu machine, this is essentially the same thing — we are adding some custom-defined hostnames that will resolve to our custom-defined IP addresses. Kubernetes needs a CRI (Container Runtime Interface) so it can run pods. CRIs are responsible for pulling images, which are then used to run containers. The CRI of my choice is called `containerd`, and that CRI doesn't read from CoreDNS like pods running in Kubernetes do — it reads `/etc/rancher/k3s/registries.yaml`. Within my cluster, I use `zot`, a self-hosted image repository where I store the images I have built for my running apps, such as the Epoka Programming Club web app. `containerd` needs to pull from `zot` in order to run apps whose images are stored in `zot`, and for that it needs to know where to find it and what its credentials are. This information is stored in `/etc/rancher/k3s/registries.yaml`, but the address used to find `zot` doesn't use the IP address of the node it's running on — it instead uses the custom hostname `registry.gentoo.lan`, and that hostname still needs to resolve to something in the end. `networking.extraHosts` is therefore used so that `registry.gentoo.lan` can resolve to the node holding `zot`.

## Security

Security is handled using a combination of configurations and processes within my homelab setup. On the NixOS side of things, my primary concerns are that no outsider can access my nodes without my permission, that my processes can't be compromised, that my secrets stay secret, and that if any layer fails, there is another layer to support it. As such, this section will be split into two parts: configurations and secrets management.

### Configurations

Configs regarding security are mainly found in `modules/security.nix`. Let's have a look:

![[Pasted image 20260909115536.png]]

It's fairly small compared to my other files so far, but each section is important.

```nix
  services.openssh = {
    enable = true;
    settings = {
      PasswordAuthentication = false;
      KbdInteractiveAuthentication = false;
      PermitRootLogin = "no";
    };
  };
```
This enables OpenSSH on my machines, which lets me access a remote shell for any maintenance work I might want to perform on my nodes or for using `nixos-rebuild-switch`. The settings defined there make it so logging in with SSH and using a password to authenticate is impossible. The only way to log in is if your machine has the correct SSH key, and stealing SSH keys is a whole lot harder than stealing some password short enough for me to remember. Root login is also not permitted, so that if anyone does manage to log in, it would have to be the `slowking` user, who needs a password to use sudo, introducing multiple layers of defense into this setup. There is one user who doesn't need a password for sudo, and that's the `deploy` user, but since only one machine has the SSH keys for that user, that makes logging in as that user a lot harder for an attacker.

```nix
  services.fail2ban = {
    enable = true;
    maxretry = 5;
  };
```

Fail2Ban sets limits on how many password authentication attempts can fail when trying to log in. Since password authentication for SSH is not allowed here, this is less about guessing passwords and more about not letting attackers try different keys in a sequence. Once 5 failed attempts have been made, the address that's trying to log in gets blocked by the firewall.

```nix
  security.apparmor.enable = true;
```

This line enables [AppArmor](https://www.apparmor.net/) which extends the traditional capabilities of Linux permissions. Even if a process is compromised, additional profile-based rule enforcement can prevent that process from doing damage.

```nix
  systemd.coredump.enable = false;
```

Disabling systemd's core dump handling prevents secrets present within the memory of processes from being leaked in log files after a process crash. Core dumps are useful for figuring out why a process crashed, but this is a trade-off worth making.

```nix
  security.pam.loginLimits = [
    { domain = "*"; type = "hard"; item = "core"; value = "0"; }
  ];
```

PAM stands for "Pluggable Authentication Modules". Linux handles session authentication through PAM. Instead of implementing their own logins, different services (SSH, sudo, etc.) can just use PAM. Systemd isn't the only part of NixOS that can produce core dumps; user sessions can as well. This line disables core dumps at the user session level, whereby:

All users (`domain = "*"`) will have a core file of size 0 (`item = "core"; value = "0"`), and this cannot be changed (`type = "hard"`).

```nix
  services.tailscale.enable = true;
```

This enables the beloved [Tailscale](https://tailscale.com/) service. Tailscale is what I use to create a VPN so that I can expose services from my machine that I want to be accessible only from my other machines. I mainly use this to SSH to my nodes from machines outside of my home network, or to view admin dashboards of my cluster accessible only by me. There will be more on Tailscale in the cluster management part.

```nix
  networking.firewall = {
    enable = true;
    checkReversePath = "loose";
    trustedInterfaces = [ "cni0" "tailscale0" ];
  };
```

It's best if I explain this configuration line by line:

- `enable = true;` - Simple enough: this just enables the firewall.
- `checkReversePath = "loose"` is a way to protect against packets whose address has been spoofed. Attackers may send packets whose source addresses do not route to anything, which might be used to consume resources. `checkReversePath` first checks if we can route to the source IP address of the packet, dropping all packets whose source is unreachable. `loose` means that, if a packet comes, then we will use all network interfaces available to determine if the source is reachable. There is another option, `strict`, which only uses the network interface the packet came from to determine if the source is reachable. So why `loose`? This setup has a lot of network interfaces (Wi-Fi, Tailscale, Kubernetes, etc.), with a lot of packets coming through between them. Setting it to `strict` might drop otherwise valid packets, and I don't want to be the one to debug that issue.
- `trustedInterfaces = [ "cni0" "tailscale0" ];` `cni0` is the Kubernetes network interface and `tailscale0` is the Tailscale network interface. This line trusts all traffic coming from those interfaces and lets them bypass normal network restrictions.

### Secrets Management

Securely storing and retrieving secrets is a crucial part of any system aiming for security. Secrets are any pieces of information which an attacker could use to harm you. Things like passwords and encryption keys should be safely stored somewhere such that only you can access them. This project takes two different approaches for managing secrets depending on the situation. Secrets used by running applications managed by Kubernetes are managed with [Vault](https://www.hashicorp.com/en/products/vault), while the lower-level NixOS part of the project uses sops + age.

My machines need various secrets to operate properly, such as sudo passwords, Wi-Fi credentials, the k3s join token, etc. These secrets must only be accessible by my host workstation PC and my nodes, while still fitting my declarative configuration philosophy. This means that these secrets must be part of my Git repository, which is publicly viewable on GitHub, while remaining hidden from attackers. This is where sops + age and sops-nix (one of the inputs in my flake.nix) come in.

`sops` stands for "Secrets OPerationS". It's a CLI editor meant for editing structured files like `YAML` or `JSON` which contain secrets. In files encrypted by sops, the keys stay in plaintext while the values are encrypted by whatever backend is configured (in my case it's age). Within the editor, values are decrypted and sit in plaintext ready to be edited by the user, and are then re-encrypted once the editing is over. This lets us store multiple secrets in one file and use structured conventions such as that of `YAML` in order to understand what a secret is used for.

`age` is a simple file encryption tool, and, as previously mentioned, is the backend `sops` uses for encrypting its content.

`sops-nix` is the tool I have chosen to use for injecting secrets into my Nix configs. NixOS turns your configuration files into other files under the `/nix/store` path (files which every user can access) after evaluation is done. Commands like `builtins.readFile`, which Nix can use to inject content from files into its configurations, are evaluated, and then the value from those files is written in plain text inside the result. This means that if your secrets were sitting in plain text inside permission-controlled files for your user, and you then read the value of those secrets into the Nix configuration, every user on that machine would be able to read your secrets under the `/nix/store` path. With `sops-nix`, I can instead provide Nix with the path of the file that *will* contain the decrypted value, which hides the actual value of the secret from `/nix/store` or anyone else trying to access my secrets. The only places secrets can be found are on the nodes themselves or on the workstation machine when the appropriate user has decrypted the files for editing purposes.

The general flow for this is:

- Write `.sops.yaml` to provide `sops` with information on where the YAML file(s) containing secrets are and who should see them
- Write the secrets for each recipient using their respective public keys
- Set up NixOS configurations for each node with `sops` and `age` properly configured
- Reference secrets in these configurations for your use cases
- Build each configuration and install it on its respective node
- Secrets are decrypted at activation (on every boot and every `nixos-rebuild switch`) within each node and can then be read securely by other processes

From this flow, it should be clear that secrets never exist in plaintext except on the nodes themselves or while being edited on the workstation machine.

Let's see how the nodes of this homelab are configured using the shared module `sops.nix`:
![[Pasted image 20261007101832.png]]

`sops.defaultSopsFile = ../secrets/cluster.yaml;` sets `cluster.yaml` as the default file from which secrets will take their values unless a secret (like a host-specific one) has a different file configured as its source.

`sops.age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];` tells `sops-nix` to use the host's SSH private key as the age identity for decryption at activation time. The SSH public key of the node should be present on the main machine used for secrets editing. In my setup, I gave each node its SSH keys when it was first installed with `nixos-anywhere --extra-files`. The correct keys need to be present before we join the node to our cluster.

`sops.secrets."k3s/token" = { };` simply sets the secret called `k3s/token` with default options. In our case, the default options include deriving the value of the secret from the `defaultSopsFile` which was set earlier. Default options also include setting the owner of the decrypted secret file to `root` with `0400` permissions.

```nix
sops.secrets."wifi/env" = {
group = if (config.users.groups ? wpa_supplicant) then "wpa_supplicant" else "root";
mode = "0440";
};
```

This does the same thing but also lets the `wpa_supplicant` group (if it exists; otherwise, it's `root`) read the file. Permissions were also set to `0440`.

`sops.secrets."slowking/hashed-password" = { neededForUsers = true; };` sets the password for the `slowking` user with the `neededForUsers` option, which decrypts the secret early and puts it in `/run/secrets-for-users` before any user is created by NixOS.

`sops.secrets."registry/password" = { };` does the same thing as the `k3s/token` line above.

The `sops.nix` file configures and sets secrets to be used cluster-wide and shared between nodes. One example of how a secret set here is used in another module is the Wi-Fi password case we've already seen:

```nix
  config.networking = {
  ...
    wireless = {
    ...
      secretsFile = config.sops.secrets."wifi/env".path;
    ...
    };
```

`sops-nix` provides the wireless configuration with the path which will contain the decrypted Wi-Fi password on node activation.

I have a demo walkthrough where I create secrets and then safely provide them to VMs that simulate my nodes in [[NixOS Secrets Management Demo]].


## Kubernetes

Here I will talk about how I'm using NixOS to set up a working Kubernetes cluster, specifically a `k3s` cluster. All of the applications running in this project sit on top of Kubernetes, and Kubernetes itself sits on top of NixOS.

`k3s`, luckily for me, is a first-class module in NixOS, meaning that setting up a k3s control plane is as easy as writing the following config:

`modules/k3s-server.nix`
![[Pasted image 20261014210515.png]]

Apart from the extra flags, the rest is actually very simple:

```nix
  services.k3s = {
    enable = true;
    role = "server";
    tokenFile = config.sops.secrets."k3s/token".path;
    ...
```

All we have to do is enable the `services.k3s` config, set the role we desire (server or agent, the former being the control plane node and the latter being a worker node), and give the path to the file that will contain the token. In our case, this is the path to the file that will contain the decrypted token after the VM has been provided with its secrets. NixOS does the rest.

The `k3s-server.nix` module is to be imported only by specific hosts that will act as control plane nodes. Host-specific Nix configs will be explained a bit later, but for now it's sufficient to know that the `cachyos` host is the sole user of this module. Despite having just one user, it exists independently of the host's module, in case other hosts wish to import it as well in the future.

Now for the `extraFlags` bit, which customizes our cluster:
```nix
    extraFlags = [
      "--flannel-backend=host-gw"
      "--node-name=cachyos"
      "--node-ip=${node.ipv4}"
      "--flannel-iface=${node.wifiInterface}"  # pin flannel to WiFi (not tailscale0/wwan)
      "--tls-san=192.168.1.31"
      "--tls-san=k3s.gentoo.lan"
      "--write-kubeconfig-mode=0644"
      # Longhorn only auto-creates its default disk on nodes carrying this label
      # (create-default-disk-labeled-nodes=true). Without it the node reports 0
      # storage and no PVC can schedule.
      "--node-label=node.longhorn.io/create-default-disk=true"
      "--secrets-encryption"
    ];
```

`--flannel-backend=host-gw` is fitting for my use case. Kubernetes needs a CNI (Container Networking Interface) plugin to work, which creates a virtual network and is responsible for communication with pods. `flannel` is a simple CNI plugin for Kubernetes. It's not the most powerful, but for small setups such as mine, it gets the job done. The default `flannel` backend is `VXLAN`, which performs a ton of work to move packets between pods. That work is necessary for complex setups, but `host-gw` avoids all of it by taking advantage of the fact that all of my nodes are within the same LAN.

```nix
      "--node-name=cachyos"
      "--node-ip=${node.ipv4}"
      "--flannel-iface=${node.wifiInterface}"
```

These three provide the node in the cluster with a name, an IP address, and the network interface `flannel` will use for communication with pods.

```nix
	  "--tls-san=192.168.1.31"
      "--tls-san=k3s.gentoo.lan"
```

The above flags deal with TLS and certificates. Whenever a client talks to a server, the client generally wants to make sure that they really are talking to the person they intend to talk to, and that what is said between them remains anonymous. The internet has solved this problem through [asymmetric encryption](https://www.geeksforgeeks.org/computer-networks/what-is-asymmetric-encryption/) and certificates. Certificates contain information about who they represent and a signature from a trusted authority confirming that the certificate is legit. Information about the host used to be just a CN (Common Name), which held the hostname of the server, but today TLS uses the SAN (Subject Alternative Name) field, which can hold a list of DNS names or IP addresses, to verify the host. When k3s boots for the first time, it generates a CA (Certificate Authority), which then issues a certificate for the API server. This generated certificate contains the following names and IP addresses:

- `localhost`, `127.0.0.1` (for in-cluster components)
- the node's auto-detected IP address and hostname
- `kubernetes.default.svc` (the in-cluster DNS name for the API)

So all clients that send requests to the server using names or IP addresses present in the certificate will trust the API server. Now consider what would happen if someone were to try to talk to `k3s.gentoo.lan`: their request would get routed to the API server, which would present a certificate whose names and IP addresses don't include `k3s.gentoo.lan`. That would lead to TLS failing between the client and the host, rendering communication impossible. The flags above add the additional name `k3s.gentoo.lan` and IP address 192.168.1.31 to the generated certificate, so clients looking to talk to a server with that name or that IP address will see a certificate they recognize.

`--write-kubeconfig-mode=0644` sets the permissions of the `kubeconfig` file, which grants full control of the cluster to anyone who has it. Normally, the default is `0600`, which lets only the root user access it, but since I'm the only person who can log into my nodes, `0644` lets anyone read it, making it convenient for my `slowking` user to control the cluster.

`--node-label=node.longhorn.io/create-default-disk=true` is a bit more complicated, as it requires cooperation with another config within the Kubernetes configs, namely the Longhorn config. In short, Longhorn provides distributed block storage, which can be used in combination with Kubernetes Persistent Volumes so that stateful workloads such as a database can get resilient and reliable storage. For this project, `create-default-disk-labeled-nodes=true` has been set for Longhorn, which will cause it to create default disks only on nodes that contain the label `node.longhorn.io/create-default-disk=true`. This line sets that label. Some nodes might not need this label (such as a compute-only node; I have many future plans), which is why default disk creation is an opt-in feature for our cluster.

`--secrets-encryption` encrypts Kubernetes secrets at rest. The default is just base64 encoding. Although I doubt I will ever need it in the future (this protects against things like stolen backups), I aim for my cluster to be as close to a production-level cluster as possible within reasonable limits, and this line is a fairly easy way to close that gap.

Now that the `k3s` configs for the server are done, we have:

`networking.firewall.allowedTCPPorts = [ 6443 10250 80 443 9100 ];` opens up the ports needed for our `k3s` server to work. `6443` is the port the kube API server uses, `10250` is what kubelet uses, `80 443` are HTTP(S) ports, and `9100` is for the `node-exporter`.

Now that the server is done, it's the agent's turn:

`modules/k3s-agent.nix`
![[Pasted image 20261014224738.png]]

As you can see, this file is pretty much the same as the other, so I'll only go over the notable differences:

- `role = "agent"` is the first and probably the most important difference: it's what makes the node that imports this module an agent that joins the server rather than a server of its own.
- `serverAddr = "https://192.168.1.31:6443";` tells the agent where to find the server, and thanks to the `--tls-san` changes we added on the server, TLS will work with `192.168.1.31`.
- There are fewer `extraFlags`: only those related to node identity, along with the Longhorn config for default disk creation.
- The kube API port is not included in `allowedTCPPorts`, since no process on agent nodes will listen on that port.

All nodes running my cluster share some configurations for k3s. Hence, a separate k3s commons module that every node gets by default from `flake.nix` was created:

`modules/k3s-common.nix`
![[Pasted image 20261023161834.png]]

Let's go through everything one at a time:

```nix
  services.openiscsi = {
    enable = true;
    name = "iqn.2020-01.io.homelab:${config.networking.hostName}";
  };
```

The block above enables the `openiscsi` service, which exposes disks from one machine to the network for anyone else to access. It's how a Longhorn volume created on one machine can be accessed from a pod on another machine. `name = "iqn.2020-01.io.homelab:${config.networking.hostName}";` gives each of the hosts a unique and stable name (stable refers to a name that stays the same even after a reboot).

`boot.kernelmodules = [ "iscsi_tcp" "br_netfilter" "overlay" "dm_crypt" ];` enables the following kernel modules for my nodes:

- `iscsi_tcp` lets disks exposed via `openiscsi` be accessible through TCP.
- `br_netfilter` makes packets pass through the rules set by `iptables`, which can be controlled by `NetworkPolicies` resources.
- `overlay` is used by container runtimes to manage filesystems. Containers will typically have a base filesystem from a given OS and then make edits to that filesystem so that the app(s) within that container can work. If you have many containers running the same base filesystem, having copies of these bases can be wasteful. `overlay` allows you to compose many filesystems by merging "lower" immutable base filesystems with "upper" edits to those base filesystems, essentially reusing the base FS without wastefully copying it.
- `dm_crypt` functions as a layer between applications and the underlying disk. Whenever an app wants to write, `dm_crypt` will encrypt the contents before the write is given to the disk. Whenever an app wants to read, `dm_crypt` will first decrypt the contents retrieved from disk before serving them to the app. It does not enable encryption by default; it only enables the capability to use it. Encrypted Longhorn volumes need this module.

`boot.supportedFilesystems = [ "nfs" ];` enables support for NFS (Network File System). `openiscsi` supports access to disks over the network, while NFS supports access to files over the network. NFS is required for RWX (ReadWriteMany) Persistent Volumes, which are volumes that can be read from and written to by more than one node.

`environment.systemPackages = [ pkgs.nfs-utils pkgs.openiscsi pkgs.cryptsetup ];`
This installs a couple of useful tools.

```nix
  systemd.tmpfiles.rules = [
    "d /usr/local/bin 0755 root root - -"
    "d /usr/local/sbin 0755 root root - -"
    "L+ /usr/local/bin/iscsiadm  - - - - /run/current-system/sw/bin/iscsiadm"
    "L+ /usr/local/sbin/iscsiadm - - - - /run/current-system/sw/bin/iscsiadm"
  ];
```

`systemd` supports managing files through declarative rules (hooray for Nix users!). The rules and their combinations are a bit lengthy to explain here. I myself don't know them; I have to Google them or ask AI to generate these rules, but I'll explain enough for you to understand what's going on here. The `systemd.tmpfiles.rules` option expects an array of strings, each of which is its own rule. A rule is created by a series of 7 arguments: Type, Path, Mode, User, Group, Age, Argument. Type refers to what we want to do. For example, `d` means "create if it does not exist". `L+` means "create a symbolic link if it does not exist". Path simply refers to the path we want to operate on. Mode sets the permissions of the path. User and Group set the owner and the group of that path. Age sets a time for when this path should be deleted, and Argument can mean different things depending on the Type supplied in the rule. `d` will always have the Argument `-`. When `L+` is present, the Argument will refer to the symlink target.

Knowing this, the above config block creates the two directories in the first two rows, and the other two rows create symbolic links from files in those directories to other directories. Packages installed using Nix usually sit under the `/nix/store` path, which means that a tool that needs a binary from a package won't find it in places where binaries are usually found in Linux. Therefore, a symbolic link for `iscsiadm` was needed.

` hardware.enableRedistributableFirmware = true;` permits the use of proprietary Wi-Fi firmware even if the installation process leaves this option out by default.

```nix
  sops.templates."registries.yaml".content = ''
    mirrors:
      registry.gentoo.lan:
        endpoint:
          - "https://registry.gentoo.lan"
    configs:
      "registry.gentoo.lan":
        auth:
          username: registry-admin
          password: ${config.sops.placeholder."registry/password"}
        tls:
          ca_file: /etc/rancher/k3s/certs/gentoo-internal-ca.crt
  '';
```

The block above creates a file, `registries.yaml`, with the specified content, which contains the credentials for `zot`. These credentials are needed by `containerd` in order to pull and use images stored within it. The contents of this file are in a format `containerd` can understand and contain the path to the file that will have the decrypted `zot` password.

```nix
  environment.etc."rancher/k3s/registries.yaml".source =
    config.sops.templates."registries.yaml".path;
```

This puts the file we created at a path `k3s` expects.

```nix
  environment.etc."rancher/k3s/certs/gentoo-internal-ca.crt".source =
    ../assets/gentoo-internal-ca.crt;
```

This sets the TLS cert for Traefik, my current Kubernetes Ingress Controller. Like every other app in my cluster, `zot` sits behind the Ingress Controller when talking to processes outside of the cluster's existing pods, and that Ingress Controller needs to present a cert for TLS to be established. `containerd` needs `zot` in order to pull images, and for that connection to occur, `containerd` must trust the proxy sitting in front of `zot`.

## Hosts

Most of what we've covered so far configures all nodes present in the cluster unless specifically stated otherwise. In this section, I will be discussing host-specific configurations and how some modules are imported for just one of the hosts. Let's have a look at the relevant section of the project for this topic:

![[Pasted image 20261027214552.png]]

Here we can clearly see a directory for each host, each of which has 3 files: `default.nix`, `disko.nix`, and `hardware.nix`. I'll cover them from least important for our purposes to most important:


`hosts/{host}/hardware.nix`
![[Pasted image 20261027220654.png]]

`imports = [ (modulesPath + "/installer/scan/not-detected.nix") ]` imports a standard NixOS module for detecting devices our `hardware.nix` might miss.

`boot.initrd.availableKernelModules = [ "nvme" "xhci_pci" "usbhid" "usb_storage" "sd_mod" ]` loads kernel modules for `initrd`, a small in-memory FS Linux needs in order to find and mount the real root FS.

```nix
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;
```

This enables the bootloader and lets NixOS manage boot entries (like the different generations) in the machine's UEFI firmware.

`hardware.cpu.amd.updateMicrocode = lib.mkDefault true` on `cachyos` and
`hardware.cpu.intel.updateMicrocode = lib.mkDefault true` on `tux` enable updates for `microcode`, low-level software that runs inside the CPU and turns x86-64 instructions into micro-operations (the instructions your CPU actually performs). This line is different for each host since they have different CPU vendors.

`nixpkgs.hostPlatform = "x86_64-linux"` tells the package manager which architecture the host machine uses so its packages will match.

`hosts/{host}/disko.nix`
![[Pasted image 20261028214359.png]]

`disko.nix` uses Disko, the third-party module previously mentioned in the entry point section of the NixOS project, to set up the physical SSDs of my nodes so that Linux can properly use them.

`disko.devices.disk.main = { ` tells Disko to configure a device of type disk called "main" (I could have called it anything; the line could very easily have been `disko.devices.disk.thanas`) using the settings I've provided for it below.

`type = "disk"` simply tells Disko we are referring to an entire physical disk.

`device = "/dev/disk/by-id/nvme-SAMSUNG_MZAL4256HBJD-00BL2_S67PNF0W869279"` tells Disko where to find the physical disk. This is different for each host since they have different physical disks. `cachyos` has the Samsung NVMe drive, while `tux` has the SATA drive we discussed earlier, and its line is `device = "/dev/disk/by-id/ata-SanDisk_X400_M.2_2280_256GB_170717803474"`.

`content = {` contains the specifications of the partition.

`type = "gpt";` specifies that [GPT](https://en.wikipedia.org/wiki/GUID_Partition_Table) partitioning, a standard modern partitioning method, is to be used.

```nix
  partitions = {
	ESP = {
	  size = "512M";
	  type = "EF00";
	  content = { type = "filesystem"; format = "vfat"; mountpoint = "/boot"; mountOptions = [ "umask=0077" ]; };
	};
	root = {
	  size = "100%";
	  content = { type = "filesystem"; format = "xfs"; mountpoint = "/"; mountOptions = [ "noatime" ]; };
	};
  };
```

This block describes the actual partitions we want. We need two in total: one to hold files for starting the computer itself, and one for the main FS everything else will run on.

ESP is just the name we are giving the first partition, and it is supposed to mean "EFI System Partition", a partition for systems that use UEFI to boot. The `size` config of `512M` specifies that the allocated space for this partition is 512 megabytes. `type= "EF00"` tells the computer that this partition is used for holding startup files. The `content` config simply tells Disko that this partition will be a `filesystem` in the `vfat` format (this is the format the startup software understands), mounted at the `/boot` path, with access restricted to root.

The other partition here is called `root`. Its size will cover 100% of the remaining space on the disk, and its contents will be those of a `filesystem` in the `xfs` format (whatever works for Longhorn works for us too in this case; Longhorn supports both `xfs` and `ext4`). It will be mounted at the `/` path, and a file's access time won't be updated every time it's read (that would put the SSD under a lot of load due to bookkeeping).

`disko.nix` is fairly straightforward; its job is only to do basic partitioning.


The most important file in this section would be the `default.nix` file. It's responsible for giving each of the hosts its own identity and role. I will cover the `default.nix` file for each host in descending order of role:

`hosts/cachyos/default.nix`
![[Pasted image 20261110152617.png]]

This file starts off by importing the configurations for laptops (`laptop.nix`) and for the k3s server (`k3s-server.nix`), which establishes that this host is going to be the server for the cluster.

`networking.hostName = "cachyos"` gives our host a recognizable name on the network; it should be different for each host.

```nix
homelab.node = {
  wifiInterface = "wlan0";
  ipv4 = "192.168.1.31";
};
```

This sets a static IP address for our host (a must for our server since we want agent nodes to easily discover it) on the `wlan0` network interface, which corresponds to the interface connected to Wi-Fi.

```nix
services.udev.extraRules = ''
KERNEL=="nvme0n1", SYMLINK+="monitored-disk"
'';
```

This line simply creates a symbolic link from `/dev/monitored-disk` to `/dev/nvme0n1`. This same setting exists on the other node as well, but the link points to a different path. Its purpose is to create a universal path for all hosts that my disk monitoring program can use without much fuss.

`system.stateVersion = "25.11";` tells NixOS to preserve compatibility with the `25.11` system version.


`hosts/tux/default.nix`
![[Pasted image 20261110153419.png]]

The files are largely similar to one another, so I'll go over the differences between them. The first and most important difference would be its imported role within the cluster: `k3s-agent.nix` makes this host an agent. This host also imports `gitea-runner-seccomp.nix`, a file containing configurations for running the Gitea runners. Its file looks like this:

```nix
{ ... }:
{
  systemd.tmpfiles.rules = [
    "d /var/lib/kubelet/seccomp 0755 root root -"
    "L+ /var/lib/kubelet/seccomp/gitea-runner-buildah.json - - - - ${../assets/gitea-runner-buildah.json}"
  ];
}
```

It's small, but it belongs in its own file as a self-contained module that hosts can conditionally import. [Gitea](https://about.gitea.com/) is a self-hosted DevOps platform used much like [GitHub](https://github.com/). Its main functions in my cluster are Git repository hosting and CI/CD. The latter functionality is achieved by running certain jobs whenever they are triggered, either manually or when a pull request is merged into the main branch of a repository. These jobs run tests on your code and, upon successful completion of the tests, execute some kind of command that updates software that's already running to use the new code that just got merged. These jobs are executed by a Gitea runner. Now, my Gitea runner uses [Buildah](https://buildah.io/) to create container images from Dockerfiles. Buildah needs to run some commands that most of my other applications are not allowed to run, so I need some kind of way to give it access to only those commands while keeping others restricted. Most operations we want to run tend to make syscalls to the Linux kernel. Which processes are allowed to make which syscalls is controlled by [seccomp](https://en.wikipedia.org/wiki/Seccomp). Within this project, I have created a file that contains a custom seccomp profile just for Buildah, listing what it can and can't do. The `gitea-runner-seccomp.nix` file then uses `systemd`'s `tmpfiles` rules in order to ensure that a `/var/lib/kubelet/seccomp` directory exists and that the seccomp profile JSON file is present within that directory. Seccomp profiles can be provided to Kubernetes when running a pod, and the `/var/lib/kubelet/seccomp` directory is where Kubernetes looks for them. Then, I specify in the runner YAML file that I want the seccomp profile called `gitea-runner-seccomp.json`, and Kubernetes fetches it for me.

All of this is to say that my `tux` node is the node of choice for hosting Gitea runners.

`networking.hostName = "tux"` sets the name for this host to `tux`.

```nix
homelab.node = {
  wifiInterface = "wlp1s0";
  ipv4 = "192.168.1.25";
};
```

This does the same thing as it does on `cachyos`, but the static IP address for `tux` is `192.168.1.25` instead of `192.168.1.31`, as it is on `cachyos`.

```nix
services.udev.extraRules = ''
  KERNEL=="sda", SYMLINK+="monitored-disk"
'';
```

This also has the same function, but the difference is in the path. The symbolic link here points to `/dev/sda`.


And that is everything for the NixOS half of my homelab!
