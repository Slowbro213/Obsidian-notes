# Secrets Management

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

The `sops.nix` file configures and sets secrets to be used cluster-wide and shared between nodes. One example of how a secret set here is used in another module is the Wi-Fi password case covered in [[Projects/homelab/NixOS/Networking|Networking]]:

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

---

← Previous: [[Projects/homelab/NixOS/Security|Security]] | Next: [[Projects/homelab/NixOS/Kubernetes Setup|Kubernetes Setup]] →

[[Projects/homelab/Nixos|NixOS Overview]]
