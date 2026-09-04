>[ Nix is a tool that takes a unique approach to package management and system configuration. Learn how to make reproducible, declarative and reliable systems.](https://nixos.org/#:~:text=Nix%20is%20a%20tool%20that%20takes%20a%20unique%20approach%20to%20package%20management%20and%20system%20configuration%2E%20Learn%20how%20to%20make%20reproducible%2C%20declarative%20and%20reliable%20systems%2E)

NixOS is an operating system built on top of the nix package manager.  It lets me declare how I want my nodes to be, down to the finest details, and then makes that declaration a reality. Many of its features make it an attractive tool for managing the configuration of your servers, such as: 

- Reliable Upgrades - Nix is a purely functional package manager, meaning that the same parameters will always yield the same result. It's particularly useful for repeatable builds.
- Atomic Upgrades - NixOS takes a transactional approach to configuration upgrades. If you want to change the config of a server of yours, and after applying the changes, for whatever reason the upgrade process fails or is interrupted, the entire upgrade attempt is aborted, and your server is in the exact same state as it was prior to the config upgrade.
- Rollbacks - Each time a new `sudo nixos-rebuild switch` is used to upgrade your configuration, its old versions are still stored on your machine. You have the ability to switch to any of those old versions. Even if you were to delete those old versions through `nix-collect-garbage -d`, the power of Git in this repository allows me to revert any commit such that my config is back to an older version, which I can then just apply.

The power of NixOS is not limited to what is mentioned here and surpasses my human understanding, but the above section should provide an idea as to why anyone would choose to use this tool.

One particularly useful feature of NixOS is [Nix Flakes](https://nixos.wiki/wiki/flakes). According to the docs:
>[Nix flakes provide a standard way to write Nix expressions (and therefore packages) whose dependencies are version-pinned in a lock file, improving reproducibility of Nix installations.](https://nixos.wiki/wiki/flakes#:~:text=Nix%20flakes%20provide%20a%20standard%20way%20to%20write%20Nix%20expressions%20%28and%20therefore%20packages%29%20whose%20dependencies%20are%20version%2Dpinned%20in%20a%20lock%20file%2C%20improving%20reproducibility%20of%20Nix%20installations%2E)

As you might have guessed, I have used Nix Flakes in this project, and it is indeed a flake that serves as the entry point. Below I'll provide the entire tree of the `nixos` directory for reference:

<img style="display: block; margin-left: auto; margin-right: auto; width: 50%;" src="Pasted image 20260813213743.png"  alt="filetree" />
# The Entry Point

Everything begins with the `flake.nix`. Usual function definitions go something along the lines of: input -> logic -> output. The contents of the `flake.nix` file are no different:


![[Pasted image 20260826224348.png]]

>[!NOTE]
>The `nixpkgs/nixos-25.11` version has been updated to pull in the newest Linux kernel.

A Nix Flake is comprised of `inputs`, `outputs` and a `description`. The description makes no difference in functionality. In `inputs` you declare what your flake needs in order to work, and in `outputs` you declare what your flake will create based on those inputs. Within this flake, I have declared that my inputs will be: the nix packages repository, `disko` ( a third-party module which allows for declarative disk layouts and partitioning ) and `sops-nix` ( a third-party module for managing secrets in my repo ). And in my outputs, I have first declared a helper function in nix called `mkNode`, which takes as an argument the `hostname` of a node and returns a `nixosSystem` configuration for that specific `hostname`. Then I have declared that my `nixosConfigurations` will be an attribute set ( think dictionaries in Python, just a set of key-value pairs ) where the values are nodes created from hostnames. As a result, every node is comprised of the same modules, only differing in those modules which are dependent on the hostname itself. This approach provides code reusability and minimizes duplication while keeping shared behavior between nodes consistent. It will be especially useful once I have more nodes, since adding a node is as trivial as adding a couple of files and modifying this attribute set.

// TODO: explain NixOS modules (e.g., NixOS comes with variables like environment.systemPackages on its own to define packages in your system)
 
# Shared modules

The entry point creates `nixosSystems` from shared modules as well as node-specific modules. Since shared modules are present on all nodes, I'll be covering them first.

## Commons

`modules/common.nix`
![[Pasted image 20260827223508.png]]

Every module gets a list of arguments available to it from the flake. The list at the top is not exhaustive, meaning any given module may have access to more arguments than declared. It's just that the only declared arguments are the relevant ones for this module. `config` is an argument that contains contributions from every module and makes those contributions available to all modules that use it. Nix doesn't evaluate modules in file order. It instead first combines every contribution to the `config` object from every module and then starts evaluation. `common.nix` in our case is using a value from config.sops.secrets, which contains the path to the hashed password of the `slowking` (me!) user. Secrets management for the NixOS part of the project will be explained in detail later on. For now, just keep in mind that `config.sops.secrets.*` provides us with different values which are safely encrypted and stored in this repository. `pkgs` contains the packages repository provided by the nix package manager, and `lib` provides utility functions for manipulation of objects in nix.

After the list of arguments, you'll find I have declared a helper list, which contains the ssh public keys of different machines I use to ssh into my nodes. I later reference this list in other configurations below. Since they are public keys, it's completely safe to have them out in the wild like this!

Below the list of arguments, different configurations for a node are provided. I'll walk through each one.

```nix
time.timeZone = "Europe/Athens";
i18n.defaultLocale = "en_US.UTF-8";
```
The above code snippet serves to set the timezone and language of the system.

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
This snippet contains configs for nix itself. The first attribute set declares that nix may use the described experimental features, can `auto-optimise-store` and has 2 trusted users: `root` and `deploy`. The deploy user is a special user who may use passwordless sudo in order to run `nixos-rebuild switch` for applications of new configurations. This part will be explained in more detail later. Below that you'll find configurations for the nix garbage collector, a crucial component of nix which ensures the cleanup of packages that are no longer used by your system. In this code snippet I'm declaring that garbage collection runs automatically every week and deletes packages that are no longer used AND are older than 14 days. The 14-day condition is there to ensure that packages from older generations of configs can still be present if I decide to roll back.

```nix
users.mutableUsers = false;
```

This line forbids the editing by hand of users and groups. The one and only way to edit users and their configurations is through nix. By keeping users immutable, I prevent someone (like a dumber me in the future) from editing users through ssh and then forgetting about it (the main issue NixOS solves in the first place).

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

Here I have configured two users who are present in all nodes. The first user is `slowking` (me!), and the second is the `deploy` user. For the `slowking` user I have specified that they are a normal user which just means that logging in and having a shell is possible for this user. They have uid 1000, which can stay unspecified. However, if it does stay unspecified, then NixOS will auto-assign a uid to that user, and if I were to add users in the future, then my slowking user might get a new random auto-assigned uid while there are still files in the system created by slowking using the old uid. Pinning it manually resolves this issue. They are part of the `wheel` group, which is a necessity for using the `sudo` command. You can log into the slowking user via ssh with any of the admin keys listed by the `adminKeys` list, and finally their password hash exists in a file the path of which is specified in sops secrets. 

The `deploy` user has much the same characteristics, except that only one public ssh key is allowed to log in as this user and they don't have a password (meaning that the only way to log in as this user is via ssh).

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

And lastly for the `common.nix` file, I have specified what system packages are available on my nodes. NixOS looks at the list of specified packages within `environment.systemPackages` and ensures your system will have them downloaded and ready to be used after applying the changes. The `with pkgs;` bit lets us omit the `pkgs.*` part before each entry in the list and makes for a cleaner-looking list. The name of each entry correlates to the name of a package within the nix package manager.

## Performance

A Linux machine can be tuned for better performance depending on your workflow. Kubernetes clusters with lots of components running tend to be quite RAM-hungry, and tuning your machine such that `zramSwap` is enabled, for example, can help in dealing with RAM-heavy moments or workloads. Let's have a look at `performance.nix` to see what was tuned for my specific use cases and why:

`modules/performance.nix`
![[Pasted image 20260831105855.png]]

This module holds many configurations, some of which are overridable per node if the need for it arises in the future. Let's have a look at the first:

```nix
  boot.kernelPackages = lib.mkDefault pkgs.linuxPackages_latest;
```

The first line is telling nix that the Linux kernel version that's going to be used is by default the latest one. `lib.mkDefault` is responsible for setting the value of `boot.kernelPackages` if and only if the value is not present already. There is a kind of ordering when declaring what value a certain config will get, and it goes as follows from lowest priority to highest:

```nix
# Lowest
x = lib.mkDefault y

# Middle
x = y

# Highest
x = lib.mkForce y
```

Setting the kernel version as a default value is what allows us to override this configuration depending on the node and what our wishes are. So far both of my nodes hold the same version, but this might not be what I want in the future. Hence, I have taken preemptive measures.

Next, we have some kernel parameters:
```nix
  boot.kernelParams = [
    "transparent_hugepage=madvise"
    "psi=1"
    "mitigations=off"
  ];
```

`transparent_hugepage=madvise` lets processes that may benefit from [THP (transparent huge pages)](https://lwn.net/Articles/359158/) use it, while defaulting to the normal page size for other processes that don't explicitly call it. THP is a feature of the Linux kernel which allows for larger OS pages. OS pages in x86_64 Linux are 4kb, but with THP they can be 2mb. THP is only really good for a couple of things, and can be horrible for many others. Linux gives each process its own virtual memory address space. Whenever a process asks for memory, it's given a range of virtual addresses to [memory pages](https://en.wikipedia.org/wiki/Page_(computer_memory)), and for a page to be used its virtual address needs to be translated into a physical address. To avoid doing this address translation for frequently accessed addresses, a small cache of virtual to physical addresses called the TLB (translation lookaside buffer) is used. Say you allocated 2gb of memory. Normally this memory buffer will be divided into 4kb memory pages, which amounts to ~500k pages in total, which cannot all fit into the TLB (~2k entries in total). Randomly accessing parts of this memory buffer (such as in the case of a hashmap) would result in continuous TLB misses and memory translation costs. Now let's say THP was enabled for our process. This would make our 2gb slab of memory fit into just 1024 memory pages, which fully fits in typical x86 TLB, speeding up our workflows. Virtual-to-physical address translation in Linux works through a 4-level radix tree, where each node in the tree has 512 children. A virtual address is made of 48 bits. Translation works by traversing this tree using the bits of the virtual address as the path. Some of the bits within the virtual address are used to find the location of the memory page, while the others are used to locate the exact memory address inside the page. Since pages are typically 4kb in size, this means we need log2(4kb) bits to store an address within the page, amounting to 12 of the bits existing just to use the page after finding it. Now we're left with 36 bits, which are then further divided into 4 sections of 9 bits each. Each section is used on each level to go further down the tree, with the 9 bits representing the number of the child to go to for each level. So, doing address translation for 4kb pages requires a traversal of 4 levels. For a 2mb page this is different: log2(2mb) results in 21 bits instead of 12, leaving 27 bits needed for tree traversal. Splitting these 27 bits into sections of 9 leaves us with 3 levels of the tree needed instead of 4, which lets us skip one level of the tree for faster overall translation. This last part is a very, very slight optimization, but worth mentioning since it's a cool part of the inner workings of CPUs and OSes.

Transparent huge pages can cause a lot of issues if misused. Whenever a process demands a page of memory, the OS has to allocate one from RAM. Allocating a 4kb memory page is significantly easier than allocating a 2mb one. As a matter of fact, allocating 512 4kb pages is easier than allocating a single 2mb page due to [memory fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)). Continuous use and misuse of huge pages can lead to periods of massive stalls where the OS is struggling to find a contiguous 2mb slab of memory, significantly slowing down your processes rather than speeding them up. Other issues occur on processes that use `fork()` and write to memory originally allocated in the parent process. Whenever a `fork()` is used, memory pages from the parent are kept as read-only for both processes. `fork()` itself doesn't do any copying. The moment a child or parent process writes to any of those pages, no matter how small the write is, and more than one process still has access to that page, the page is fully copied and handed to the process to do as it pleases. For 4kb that copy is not a huge deal, whereas a 2mb huge page is a huge deal and may slow down or overconsume RAM quite dramatically.

`psi=1` enables "Pressure Stall Information", which lets tools like the node-exporter scrape information about which processes are stalled waiting for resources and by how much. Useful when debugging why a service might be slow on the lower-level side of things.

`mitigations=off` disables a security feature of CPUs meant to protect them from speculative execution attacks. CPU performance can be a real bottleneck in my `tux` node. Slightly less of a bottleneck in my `cachyos` node, but I've found it helpful nonetheless. The security impact of this is minimal in my case since the only workloads I'm running are the ones I create.

Next, we have some power and performance settings:
```nix
  powerManagement.enable = true;
  powerManagement.cpuFreqGovernor = "performance";
  services.power-profiles-daemon.enable = lib.mkForce false;
  services.tlp.enable = lib.mkForce false;
```

All of these features do more or less the same. My nodes are laptops and as such they don't require max throughput and power all day every day. Most OSs running on laptops tend to have settings enabled which limit the performance of the laptop in order to save some battery life. My nodes have been repurposed to servers and are plugged in at all times. Therefore, these settings are only a hindrance for my use case.

Now for some RAM and swap tuning:
```nix
  zramSwap = {
    enable = true;
    algorithm = "zstd";
    memoryPercent = 50;
  };
```

Swap memory is what Operating Systems tend to do when they run out of RAM. If the demand for RAM doesn't meet supply, your OS offloads some of the RAM that isn't being actively used to a disk partition specifically for swap. The moment that data is needed again, the OS loads it back into RAM, offloading some other part to swap in the case that there still isn't enough space. `zramSwap` is similar, but instead of offloading to disk, your OS compresses some memory pages and holds that compressed data in RAM, decompressing it when that data is demanded again. These methods of swap aren't mutually exclusive, and in fact, Windows 11 uses both by default. In my case I have opted in to using just `zramSwap` instead of utilizing the disk for it. Both methods have their pros and cons. Traditional swap memory can hold a lot of data. Disk memory is less expensive and bigger than volatile memory, but its speed is limited by your hard disk. Even SSDs can be slow when there's lots of I/O demand from other processes. `zramSwap`, on the other hand, trades some capacity for speed. The CPU is quite fast at compression and decompression, significantly faster than your disk is at reading and writing, but compression can only save so much space. It can be especially ineffective at compressing seemingly random data such as in the case of encrypted data. Nonetheless, the swap mechanism is a necessity in any workload where sudden short-lived spikes in RAM demand which supersede your supply tend to occur.

The `zstd` algorithm tends to be the slower option, but it saves the most data. Other algorithms like `lz4` and `lzo` are faster but compress worse. `memoryPercent` defines the maximum amount of RAM `zramSwap` is allowed to use for holding compressed data within RAM itself. As I've come to find out, most people recommend 50% as a good default value.

Now for some kernel runtime parameters tuning:
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

`vm.swappiness` is well above the default value of `60`. This number refers to how likely the kernel is to use swap memory as opposed to loading a page back to RAM from swap. In most cases where swap uses the disk, you want this value to be at default or lower, since offloading RAM often can be slow. But here we are using `zramSwap`, which can load and offload memory at significantly higher speeds than the disk can, making swap memory usage something we want to actively encourage for better memory management.

`vm.overcommit_memory` lets processes allocate more memory than is available on the machine. Most arena allocators tend to double their capacities over time, which can result in some huge memory demands that can be very hard to meet while barely utilizing all of the demanded memory anyway. Arena memory allocators are common in high performance applications such as databases or something like the JVM. Overcommitting memory is possible due to virtual memory. Whenever a process initially asks for memory, the OS just marks that memory as allocated virtually. Actual RAM is only ever allocated when demand is present. The OS lazily maps virtual pages to physical ones during a process's lifetime. So if your machine has 8GB of RAM, and a process asks for a 10GB slab of memory, the OS only allocates RAM for the parts of that 10GB virtual memory the process actually uses. `fork()` can also cause this due to duplicating the virtual address space of the parent, even though the child might not use all of it.

`vm.max_map_count` controls how many `mmap()` regions are allowed for a single process. Current demand does not justify an increase from the default level, but my future plans will probably match this.

`vm.dirty_background_ratio` and `vm.dirty_ratio`: these two configs are similar and work together. Whenever something is written to a file, the OS doesn't outright flush that write to disk. Instead it keeps that write in memory pages and flushes them when appropriate. `vm.dirty_background_ratio` sets the percentage of RAM the OS can use before flushing pages to disk gradually and asynchronously. `vm.dirty_ratio` is similar, but instead of writes having to be async, they become blocking. No more RAM is allowed to be used for file writes than what this config is set to, and it will block future file writes until the existing pages are flushed out. For critical workloads where data loss cannot be tolerated, keeping this number low can be beneficial (for example, WALs which don't fsync right away may lose more data if the number is too big). 

`fs.inotify.max_user_instances`: inotify is used by processes that watch files for changes. Kubernetes uses this a lot, and the default value for this parameter tends to be too low.

`fs.file-max` controls how many file descriptors are allowed to exist at once. My nodes currently run lots of pods and will only run more pods in the future, causing an increase in file descriptors. Better to make some headroom now than having to debug pods crashing from something I'd never think to check.

`fs.aio-max-nr` refers to the total maximum number of async I/O requests processes allowed to exist in this machine. There really is no limit to how high the number can be, and setting the limit to a high number can help processes which make a lot of async I/O requests such as databases. The only reason to set a non massive number is to prevent misbehaving processes from exhausting all kernel memory by making unlimited I/O requests.

`net.ipv4.ip_forward` when enabled allows machines to route packets between interfaces. A requirement for pod-to-pod communication in Kubernetes.

`net.bridge.bridge-nf-call-iptables` is also required to be enabled for Kubernetes. `NetworkPolicies` specifically. Pod network is bridged traffic, and this setting lets `iptables` control bridged traffic, which `NetworkPolicies` can control. 

`net.core.somaxconn` impacts how TCP sockets behave. Whenever an incoming TCP connection arrives in a socket, it sits in a queue until that connection is `accept()`-ed. This config controls how big that queue can be per TCP socket. Setting it to a high number can help with not dropping connections under load. Note that this is just a ceiling. Processes can still choose to have smaller queues.

`net.netfilter.nf_conntrack_max` sets the maximum number of network connections the kernel can keep track of. [Netfilter](https://www.netfilter.org/) is what the kernel uses to filter packets as well as hold information about network connections. Data about the connections includes source/destination ip/port of packets, and also NAT-related data such as what the packet's source/destination was rewritten to or from. Kubernetes specifically puts this connection tracking under a lot of load due to how `kube-proxy` manages the `Service` resource. The value that was set for this configuration allows for a lot of headroom. Errors from a misconfigured number can be a pain to debug.

`net.ipv4.tcp_congestion_control`, `net.core.default_qdisc`, `boot.kernelModules = [ "tcp_bbr" ]` are configs that work together. `boot.kernelModules = [ "tcp_bbr" ]` lets the Linux kernel see a certain congestion control algorithm, the algorithm for deciding how to send packets as fast as possible without having them be dropped by your router, called BBR as a usable option in other configs.  `net.ipv4.tcp_congestion_control = "bbr"` uses the algorithm that was loaded by the previous setting. `net.core.default_qdisc` is a configuration for the layer sitting between the kernel's TCP/IP stack and the network interface driver. It's usually the case that more than one socket is ready to transmit packets into the wire, but only some do at a time. `qdisc` decides which one, and in my case the algorithm behind it is `fq` (fair queue).  The default congestion control algorithm for Linux is `cubic`, which works by continuously sending packets faster and faster until your packets start to get dropped. Once the drops happen, it sees that as the signal to cap its rate. This algorithm rests on the assumption that packets you send are only ever dropped due to congestion, which in cases like nodes being connected over Wi-Fi, it's not always the case whatsoever, and may cap your speeds for no reason other than something unexpected happened which dropped packets. `bbr`, on the other hand, doesn't measure through loss, instead it relies on the bandwidth and RTT of the network, and uses those factors to do congestion control. As for `qdisc`, the default algorithm for Linux is just FIFO: the first packets to be queued are the first to be sent. `fq`, on the other hand, has a separate queue per network flow and has the ability to send packets at a specific time. `bbr` needs these capabilities in order to properly space out packets according to its calculations.