# Networking

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

This snippet uses the option we defined at the beginning, namely `cfg.ipv4`. Here we are defining the addresses for the network interface `${cfg.wifiInterface}` as `[ { address = cfg.ipv4; prefixLength = 24; } ]`. So there is just one address, whose value is taken from `cfg.ipv4`, which is a value set by each node itself (more on that in [[Projects/homelab/NixOS/Hosts|Hosts]]).

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

---

← Previous: [[Projects/homelab/NixOS/Laptop Configuration|Laptop Configuration]] | Next: [[Projects/homelab/NixOS/Security|Security]] →

[[Projects/homelab/Nixos|NixOS Overview]]
