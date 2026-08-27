>[ Nix is a tool that takes a unique approach to package management and system configuration. Learn how to make reproducible, declarative and reliable systems.](https://nixos.org/#:~:text=Nix%20is%20a%20tool%20that%20takes%20a%20unique%20approach%20to%20package%20management%20and%20system%20configuration%2E%20Learn%20how%20to%20make%20reproducible%2C%20declarative%20and%20reliable%20systems%2E)

NixOS is an operating system built on top of the nix package manager.  It lets me declare how i want my nodes to be down to the finest details, and then makes that declaration a reality. Many of its features makes it an attractive tool for managing the configuration of your servers, such as: 

- Reliable Upgrades - Nix is a purely functional package manager, meaning that the same parameters will always yield the same result. Its particularly useful for repeatable builds.
- Atomic Upgrades - NixOS takes a transactional approach to configuration upgrades. If you wanted to change the config of a server of yours, and after applying the changes, for whatever reason the upgrade process fails or is interrupted, the entire upgrade attempt is aborted, and your server is in the exact same state as it was prior to the config upgrade.
- Rollbacks - Each time a new `sudo nixos-rebuild switch` is used to upgrade your configuration, its old versions are still stored in your machine. You have the ability to switch to any of those old versions. Even if you were to delete those old versions through `nix-collect-garbage -d`, the power of Git in this repository allows me to revert any commit such that my config is back to an older version, which then i can just apply.

The power of NixOS is not limited to what is mentioned here and surpasses my human understanding, but the above section should provide an idea as to why anyone would choose to use this tool.

One particularly useful feature of NixOS are the [Nix Flakes](https://nixos.wiki/wiki/flakes). According to the docs:
>[Nix flakes provide a standard way to write Nix expressions (and therefore packages) whose dependencies are version-pinned in a lock file, improving reproducibility of Nix installations.](https://nixos.wiki/wiki/flakes#:~:text=Nix%20flakes%20provide%20a%20standard%20way%20to%20write%20Nix%20expressions%20%28and%20therefore%20packages%29%20whose%20dependencies%20are%20version%2Dpinned%20in%20a%20lock%20file%2C%20improving%20reproducibility%20of%20Nix%20installations%2E)

As you might have guessed, i have used Nix Flakes in this project and it is indeed a flake that serves as the entry point. Below ill provide the entire tree of the `nixos` directory for reference:

<img style="display: block; margin-left: auto; margin-right: auto; width: 50%;" src="Pasted image 20260813213743.png"  alt="filetree" />
# The Entry Point

Everything begins with the `flake.nix`. Usual function definitions go something along the lines of: input -> logic -> output. The contents of the `flake.nix` file are no different:


![[Pasted image 20260826224348.png]]

A Nix Flake is comprised of `inputs`, `outputs` and a `description`. The description makes no difference in functionality. In `inputs` you declare what your flake needs in order to work, and in `output` you declare what your flake will create based on those inputs.  Within this flake, i have declared that my inputs will be: the nix packages repository , `disko` ( a third party module which allows for declarative disk layouts and partitioning ) and `sops-nix` ( third party module for managing secrets in my repo ). And in my outputs, i have first declared a helper function in nix called `mkNode`, which takes as an argument the `hostname` of a node and returns a `nixosSystem` configuration for that specific `hostname`. Then i have declared that my `nixosConfigurations` will be an attribute set ( think dictionaries in python, just a set of key value pairs ) where the values are nodes created from hostnames. As a result every node is comprised of the same modules, only differing in those modules which are dependent on the hostname itself. This approach provides code reusability and minimizes duplication while keeping shared behavior between nodes consistent. It will be especially useful once i have more nodes, since adding a node is as trivial as adding a couple of files and modifying this attribute set.

 
# Shared modules

The entry point creates `nixosSystems` from shared modules as well as node specific modules. Since shared modules are present on all nodes, i'll be covering them first.


`modules/common.nix`
![[Pasted image 20260827223508.png]]

Every module gets a list of arguments available to them from the flake. The list at the top is not exhaustive, meaning any given module may have access to more arguments than declared, its just that the only declared arguments are the relevant ones for this module. `config` is an argument that contains contributions from every module and makes those contributions available to all modules that use it. Nix doesnt evaluate modules in file order, it instead first combines every contribution to the `config` object from every module and then starts evaluation. `common.nix` in our case is using a value from config.sops.secrets, which contains the path to the hashed password of the `slowking` (me!) user. Secrets management for the NixOS part of the project will be explained in detail later on, for now just keep in mind that `config.sops.secrets.* `provides us with different values which are safely encrypted and stored in this repository.

After the list of arguments, you'll find i have declared a helper list, which contains the ssh public keys of different machines i use to ssh into my nodes. I later reference this list in other configurations below. Since they are public keys, its completely safe to have them out in the wild like this!

Below the list of arguments, different configurations for a node are provided. Ill walk through each one

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
This snippet contains configs for nix itself. The first attribute set declares that nix may use the described experimental features, can `auto-optimise-store` and has 2 trusted users : `root` and `deploy`. The deploy user is a special user who may use passwordless sudo in order to run `nixos-rebuild switch` for applications of new configurations. This part will be explained in more detail later. Below that youll find configurations for the nix garbage collector, a crucial component of nix which ensures the clean up of packages that are no longer used by your system. In this code snippet im declaring that garbage collection runs automatically every week and deletes packages that are no longer used AND are older than 14 days. The 14 day condition is there to ensure that packages from older generations of configs can still be present if i decide to roll back.

```nix
users.mutableUsers = false;
```

This line forbids the editing by hand of users and groups.  The one and only way to edit users and their configurations is through nix. By keeping users immutable, i prevent someone (like a dumber me in the future) from editing users through ssh and then forgetting about it (the main issue NixOS solves in the first place).

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
