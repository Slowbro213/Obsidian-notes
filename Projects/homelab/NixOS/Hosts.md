# Hosts

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

`disko.nix` uses Disko, the third-party module previously mentioned in [[Projects/homelab/NixOS/Entry Point|Entry Point]], to set up the physical SSDs of my nodes so that Linux can properly use them.

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

---

← Previous: [[Projects/homelab/NixOS/Kubernetes Setup|Kubernetes Setup]]

[[Projects/homelab/Nixos|NixOS Overview]]
