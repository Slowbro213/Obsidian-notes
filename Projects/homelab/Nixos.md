>[ Nix is a tool that takes a unique approach to package management and system configuration. Learn how to make reproducible, declarative and reliable systems.](https://nixos.org/#:~:text=Nix%20is%20a%20tool%20that%20takes%20a%20unique%20approach%20to%20package%20management%20and%20system%20configuration%2E%20Learn%20how%20to%20make%20reproducible%2C%20declarative%20and%20reliable%20systems%2E)

NixOS is an operating system built on top of the nix package manager.  It lets me declare how i want my nodes to be down to the finest details, and then makes that declaration a reality. Many of its features makes it an attractive tool for managing the configuration of your servers, such as: 

- Reliable Upgrades - Nix is a purely functional package manager, meaning that the same parameters will always yield the same result. Its particularly useful for repeatable builds.
- Atomic Upgrades - NixOS takes a transactional approach to configuration upgrades. If you wanted to change the config of a server of yours, and after applying the changes, for whatever reason the upgrade process fails or is interrupted, the entire upgrade attempt is aborted, and your server is in the exact same state as it was prior to the config upgrade.
- Rollbacks - Each time a new `sudo nixos-rebuild switch` is used to upgrade your configuration, its old versions are still stored in your machine. You have the ability to switch to any of those old versions. Even if you were to delete those old versions through `nix-collect-garbage -d`, the power of Git in this repository allows me to revert any commit such that my config is back to an older version, which then i can just apply.

The power of NixOS is not limited to everything mentioned here and surpasses my human understanding, but the above section should provide a clear picture as to why anyone would choose to use this tool.

One particularly useful feature of NixOS are the [Nix Flakes](https://nixos.wiki/wiki/flakes). According to the docs:
>[Nix flakes provide a standard way to write Nix expressions (and therefore packages) whose dependencies are version-pinned in a lock file, improving reproducibility of Nix installations.](https://nixos.wiki/wiki/flakes#:~:text=Nix%20flakes%20provide%20a%20standard%20way%20to%20write%20Nix%20expressions%20%28and%20therefore%20packages%29%20whose%20dependencies%20are%20version%2Dpinned%20in%20a%20lock%20file%2C%20improving%20reproducibility%20of%20Nix%20installations%2E)

As you might guess, i have used Nix Flakes in this project. Below ill provide the entire tree of the `nixos` directory:

<img style="display: block; margin-left: auto; margin-right: auto; width: 50%;" src="Pasted image 20260813213743.png"  alt="filetree" />
# The Entry Point

Everything begins with the `flake.nix`. Usual function definitions usually go something along the lines of: input -> logic -> output. The contents of the `flake.nix` file are no different:


![[Pasted image 20260826224348.png]]

A Nix Flake is comprise of `Inputs` and `Outputs`. The description at the very top makes no difference in functionality.

 

