# Kubernetes Setup

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

The `k3s-server.nix` module is to be imported only by specific hosts that will act as control plane nodes. Host-specific Nix configs are explained in [[Projects/homelab/NixOS/Hosts|Hosts]], but for now it's sufficient to know that the `cachyos` host is the sole user of this module. Despite having just one user, it exists independently of the host's module, in case other hosts wish to import it as well in the future.

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

---

← Previous: [[Projects/homelab/NixOS/Secrets Management|Secrets Management]] | Next: [[Projects/homelab/NixOS/Hosts|Hosts]] →

[[Projects/homelab/Nixos|NixOS Overview]]
