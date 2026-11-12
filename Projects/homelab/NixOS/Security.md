# Security

Security is handled using a combination of configurations and processes within my homelab setup. On the NixOS side of things, my primary concerns are that no outsider can access my nodes without my permission, that my processes can't be compromised, that my secrets stay secret, and that if any layer fails, there is another layer to support it. This chapter covers security configurations; secrets management is covered in [[Projects/homelab/NixOS/Secrets Management|Secrets Management]].

## Configurations

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

---

← Previous: [[Projects/homelab/NixOS/Networking|Networking]] | Next: [[Projects/homelab/NixOS/Secrets Management|Secrets Management]] →

[[Projects/homelab/Nixos|NixOS Overview]]
