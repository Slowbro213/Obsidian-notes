# Common Configuration

The [[Projects/homelab/NixOS/Entry Point|Entry Point]] creates `nixosSystems` from shared modules as well as node-specific modules. Since shared modules are present on all nodes, I'll be covering them first.

`modules/common.nix`
![[Pasted image 20260827223508.png]]

Every module gets a list of arguments available to it from the flake. The list at the top is not exhaustive, meaning any given module may have access to more arguments than are declared. It's just that the only declared arguments are the relevant ones for this module. `config` is an argument that contains contributions from every module and makes those contributions available to all modules that use it. Nix doesn't evaluate modules in file order. It instead first combines every contribution to the `config` object from every module and then starts evaluation. In our case, `common.nix` is using a value from config.sops.secrets, which contains the path to the hashed password of the `slowking` (me!) user. Secrets management for the NixOS part of the project is explained in detail in [[Projects/homelab/NixOS/Secrets Management|Secrets Management]]. For now, just keep in mind that `config.sops.secrets.*` provides us with different values which are safely encrypted and stored in this repository. `pkgs` contains the packages repository provided by the Nix package manager, and `lib` provides utility functions for manipulating objects in Nix.

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

---

← Previous: [[Projects/homelab/NixOS/Entry Point|Entry Point]] | Next: [[Projects/homelab/NixOS/Performance|Performance]] →

[[Projects/homelab/Nixos|NixOS Overview]]
