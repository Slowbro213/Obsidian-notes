# Laptop Configuration

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

---

← Previous: [[Projects/homelab/NixOS/Performance|Performance]] | Next: [[Projects/homelab/NixOS/Networking|Networking]] →

[[Projects/homelab/Nixos|NixOS Overview]]
