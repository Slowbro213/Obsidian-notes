# Entry Point

Everything begins with `flake.nix`. Usual function definitions go something along these lines: input -> logic -> output. The contents of the `flake.nix` file are no different:


![[Pasted image 20260826224348.png]]

>[!NOTE]
>The `nixpkgs/nixos-25.11` version has been updated to pull in the newest Linux kernel.

A Nix flake consists of `inputs`, `outputs`, and a `description`. The description makes no difference in functionality. In `inputs`, you declare what your flake needs in order to work, and in `outputs`, you declare what your flake will create based on those inputs. Within this flake, I have declared that my inputs will be the Nix packages repository, `disko` (a third-party module that allows for declarative disk layouts and partitioning), and `sops-nix` (a third-party module for managing secrets in my repo). And in my outputs, I have first declared a helper function in Nix called `mkNode`, which takes as an argument the `hostname` of a node and returns a `nixosSystem` configuration for that specific `hostname`. Then I have declared that my `nixosConfigurations` will be an attribute set (think dictionaries in Python: just a set of key-value pairs) where the values are nodes created from hostnames. As a result, every node consists of the same modules, differing only in those modules that depend on the hostname itself. This approach provides code reusability and minimizes duplication while keeping shared behavior between nodes consistent. It will be especially useful once I have more nodes, since adding a node is as trivial as adding a couple of files and modifying this attribute set.

---

← Previous: [[Projects/homelab/Nixos|NixOS Overview]] | Next: [[Projects/homelab/NixOS/Common Configuration|Common Configuration]] →

[[Projects/homelab/Nixos|NixOS Overview]]
