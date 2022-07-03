---
id: nix-darwin-introduction
title: Declarative macOS Configuration Using nix-darwin And home-manager
created: 2022-07-03
updated: 2022-07-03
tags:
  - nix
  - nix-darwin
  - macos
  - home-manager
---

Last december, I bought myself an 14" M1 Pro MacBook Pro after being a linux user for eight years (the last few using NixOS on all my computers/servers and even my router).

Coming from NixOS, I wanted to replicate the godly feeling of declaratively configuring the entire system in every last detail with nix flakes on macOS.

[nix-darwin](https://github.com/LnL7/nix-darwin) and [home-manager](https://github.com/nix-community/home-manager) are the tools to archive exactly that.

> If I didn't knew nix-darwin existed when I needed a new laptop, I would've bought myself a framework.

## Why should anybody want to do that

- Reproducibility
  - My entire configuration is a [git repository](https://github.com/thexyno/nixos-config), on a new computer I just have to `darwin-rebuild` to get my it just as I like it.
- Reusability
  - I can just import my shell/editor/tmux configuration into both my laptop and my servers. Everything feels just like it should
- The ability to know what I have installed:
  - No 1000 forgotten packages filling up precious storage space

## Explanation of the nix ecosystem

### Nix/nixpkgs

Nix is a declarative functional programming language created as part of and for the Nix package manager.

In Nix, packages are defined as so called [derivations](https://nixos.org/manual/nix/stable/expressions/derivations.html).
A derivation is a function with package dependencies and package configuration as its inputs, and a plan on how to build that package as its output.

> package configuration in the sense of "do you want to build vlc with chromecast support?" (similar to gentoo USE flags)

Nix will evaluate that function and build it's output. The output will then be stored in the path `/nix/store/${sha256 of the derivation}-${package name}`

In [nixpkgs](https://github.com/NixOS/nixpkgs), the central repository for:

- most packages
- a library full of helper functions that make writing nix much more enjoyable
- and NixOS,

there are helper functions to easily build such a derivation for most programming languages.

> As the naming of nix/nixpkgs can get confusing:
>
> there are: nix (Programming Language), nix (Package manager), nixpkgs (Package Repository), NixOS (Linux Distribution)
>
>
> nix (Programming language) ≠ nix (Package manager). nix (Package Manager) is the only implementation of nix (Programming Language) though.
>
> nixpkgs ≠ NixOS, but NixOS ⊂ nixpkgs
>
> nixpkgs ≠ nix
>
> nix flakes ∈ nix ∧ nix flakes ∉ nixpkgs

A simple derivation is the following:

```nix
{ 
  stdenv,
  gcc,
  ...
}:
stdenv.mkDerivation { # stdenv.mkDerivation is a nixpkgs helper function
  name = "hello-1.0.0"; # Package name

  # source files for the package
  # (can also be a git repository, a flake input, ...)
  src = ./src;

  # inputs the builder needs to have
  # (we actually don't have to specify gcc, as a C compiler is part of the stdenv)
  nativeBuildInputs = [ gcc ];

  # tells nix to run gcc
  buildPhase = ''
    gcc hello.c -o ./hello
  '';

  # tells nix to copy the binary to the right path
  installPhase = ''
    # $out is the output path
    mkdir -p "$out/bin"
    cp ./hello "$out/bin/"
  '';
}
```

> in nix, a function definition is `function = argument: output`.
>
> to evaluate a function, you have to `function argument`

In the above file, we define a function with the attribute set (an attribute set is basically an object) `{ stdenv, gcc, ... }` as it's argument and `stdenv.mkDerivation {...}` as it's output.

Our function will then evaluate the function `stdenv.mkDerivation` with the attribute set after it. `mkDerivation` creates a derivation out of the attribute set, and nix will build it.

> the three dots in `{stdenv, gcc, ...}` mean, that we'll also accept an attribute set with more attributes, those will then be ignored.

### nix flakes

Normally nix uses so called channels to declare what version of nixpkgs you're running.

This is state external to your own package though, and it means you always have to look at multiple places to reproduce your exact configuration somewhere else.

To solve this problem, tools like [niv](https://github.com/nmattia/niv) and [nix-flakes](https://nixos.org/manual/nix/unstable/command-ref/new-cli/nix3-flake.html) were created.

> We'll use nix-flakes here, as it's part of nix itself
>
> **nix-flake is an experimental feature though, so be warned**

A nix flake consists of two files:

*flake.nix*

The file where all external inputs and all outputs of your flake are defined.
For example, for a simple C program, your only input will be `nixpkgs` (as `stdenv.mkDerivation` and a C compiler are inside there), and your only output will be the derivation of your program.


*flake.lock*

A JSON managed by nix itself, where the exact git hash of each of your inputs will be stored. Pretty comparable to how a `package.lock` works on npm.



A sample flake building our little C program would look like this:

```nix
{
  description = "My first nix flake";

  inputs.nixpkgs.url = github:NixOS/nixpkgs/nixos-22.05;

  outputs = { self, nixpkgs }: { # function with our inputs

    packages.default.aarch64-darwin =
      let
        pkgs = import nixpkgs { system = "aarch64-darwin"; }; # gets ourself an instance of nixpkgs
      in
      pkgs.stdenv.mkDerivation {
        pname = "hello";
        src = ./src;
        # the stdenv C compiler is at $CC (it's gcc on linux and clang on mac)
        buildPhase = "$CC -o hello ./hello.c";
        installPhase = "mkdir -p $out/bin; install -t $out/bin hello";
      };

  };
}
```

To build our flake, run `nix build .` 

> `.` means the flake at your CWD
>
> Important note: if your flake is inside a git repository, nix will ignore uncommited files

### NixOS

NixOS is a complete linux distribution based on nix.

Instead of installing and configuring packages by hand, the complete system configuration is a function with the nixpkgs and your configuration as their inputs and a script to set the new system state (the activation script) as it's output. (the output is more complicated then just the activation script, but we'll ignore that for this post)

> It dosen't have to be just the activation script though, you can also tell nix to create a iso/img/qcow2/ami of your configuration

This makes NixOS (almost) completely reproducible.

> to get all of your applications/configuration onto a new computer you just have to run `nixos-rebuild` on it and you're done (except copying over any data, but you get the point)

It also allows you to rollback to your last configuration if anything goes wrong.

### nix-darwin

nix-darwin ported the concept (and much of the code) of NixOS to macOS, so you can configure most of your mac with nix.

### home-manager

home-manager is similar to nix-darwin, but it "just" manages programs/configuration inside your users home directory.
It is (mostly) cross compatible between NixOS, other linux distributions and macOS.

## some setup

**Before doing any of this, please make sure you have a backup!!!**


`nix` has to be installed to use any of this, so please follow the instructions [here](https://nixos.org/download.html#nix-install-macos) to install `nix`.

## mac configuration

We'll need a flake to put all our configuration into. So create a new git repository somewhere and run `nix flake init` in there.

To use `nix-darwin` and `home-manager` we first have to add their respective inputs and add them to the arguments of our output function.

```nix
{
  description = "My first nix flake";

  inputs = {
      nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-22.05-darwin";
      home-manager.url = "github:nix-community/home-manager";
      home-manager.inputs.nixpkgs.follows = "nixpkgs";
      # nix will normally use the nixpkgs defined in home-managers inputs, we only want one copy of nixpkgs though
      darwin.url = "github:lnl7/nix-darwin";
      darwin.inputs.nixpkgs.follows = "nixpkgs"; # ...
  };
  
  # add the inputs declared above to the argument attribute set
  outputs = { self, nixpkgs, home-manager, darwin }: {
    # we want `nix-darwin` and not gnu hello, so the packages stuff can go
  };
}
```

To actually have a nix-darwin configuration, we'll need a `darwinConfiguration` output.

So let's add one

```nix
outputs = { self, nixpkgs, home-manager, darwin }: {

  darwinConfigurations."YourHostName" = darwin.lib.darwinSystem {
  # you can have multiple darwinConfigurations per flake, one per hostname

    system = "aarch64-darwin"; # "x86_64-darwin" if you're using a pre M1 mac
    modules = [ ./hosts/YourHostName/default.nix ]; # will be important later
  };
};
```

I like to have a folder for each computer inside of the flake for system specific configuration.
So let's `mkdir -p hosts/YourHostName` and create a `default.nix` there.

```nix
# hosts/YourHostName/default.nix
{ pkgs, ... }:
{

  # Make sure the nix daemon always runs
  services.nix-daemon.enable = true;
  # Installs a version of nix, that dosen't need "experimental-features = nix-command flakes" in /etc/nix/nix.conf
  services.nix-daemon.package = pkgs.nixFlakes;
  
  # if you use zsh (the default on new macOS installations),
  # you'll need to enable this so nix-darwin creates a zshrc sourcing needed environment changes
  programs.zsh.enable = true;
  # bash is enabled by default

}
```

If you used NixOS before, this file is like a NixOS `configuration.nix`, just with fewer and partly different options.

> You can find all options in the [documentation](https://daiderd.com/nix-darwin/manual/index.html#sec-options)

A few options I want to make sure everybody using `nix-darwin` knows of are `homebrew` and `system.defaults`.

### homebrew

The `homebrew` module lets you install software from [brew.sh](https://brew.sh) declaratively, which makes it much less a chore to deal with.

Using `nix` and something like `homebrew` or `macports` together it kinda required IMO, because most GUI applications for mac aren't inside nixpkgs (as nixpkgs can't legally ship Xcode, which is needed for mac native GUI stuff)

> The module just runs a system installed `brew` inside the activation script (meaning you'll have to install homebrew beforehand)

So if you want to install your favorite casks, you can just

```nix
# hosts/YourHostName/default.nix - inside the returning attribute set
homebrew = {
  enable = true;
  autoUpdate = true;
  # updates homebrew packages on activation,
  # can make darwin-rebuild much slower (otherwise i'd forget to do it ever though)
  casks = [
    "hammerspoon"
    "amethyst"
    "alfred"
    "logseq"
    "discord"
    "iina"
  ];
};
```

inside your host configuration

### system.defaults

The `system.defaults` module allows you to set macOS settings.
(defaults(1) is the macOS/NeXTStep central user configuration system, like GNOME dconf or the Windows Registry)

For example you can set `system.defaults.dock.autohide = true;` to autohide the dock.

> All supported options are ofc. listed inside the [options documentation](https://daiderd.com/nix-darwin/manual/index.html#opt-system.defaults..GlobalPreferences.com.apple.sound.beep.sound)

## home-manager

I wanted to use `nix-darwin` together with `home-manager`, so I can code-share between my servers and my laptop.
You don't have to do this though.
If your only computer is your mac, just use nix-darwin by itself.

To use it, we'll have to add the `home-manager` module to our `darwinConfiguration`

```nix
# flake.nix
{

  # ...

  outputs = { self, nixpkgs, home-manager, darwin }: {

    # Deleted the `packages` and `defaultPackage` outputs, as they aren't needed
    
    darwinConfiguration."YourHostName" = darwin.lib.darwinSystem {
      system = "aarch64-darwin"; # "x86_64-darwin" if you're using a pre M1 mac
      modules = [
        home-manager.darwinModules.home-manager
        ./hosts/YourHostName/default.nix
      ]; # will be important later
    };

  };
}
```

Now we can use `home-manager` in our system configuration, just like on NixOS

```nix
# hosts/YourHostName/default.nix - inside the returning attribute set
home-manager.useGlobalPkgs = true;
home-manager.useUserPackages = true;
home-manager.users.YourUserName = { pkgs, ... }: {

  stateVersion = "22.05"; # read below

  programs.tmux = { # my tmux configuration, for example
    enable = true;
    keyMode = "vi";
    clock24 = true;
    historyLimit = 10000;
    plugins = with pkgs.tmuxPlugins; [
      vim-tmux-navigator
      gruvbox
    ];
    extraConfig = ''
      new-session -s main
      bind-key -n C-a send-prefix
    '';
  };
};
```

The `stateVersion` attribute is described [here](https://search.nixos.org/options?channel=22.05&show=system.stateVersion&from=0&size=50&sort=relevance&type=packages&query=system.stateVersion)


> All the home-manager options are listed in their [documentation](https://nix-community.github.io/home-manager/options.html)
>
> home-manager `services` only work on linux though (as they use systemd, which is a linux only thing)

## Installation

To install our newly created `nix-darwin` configuration, we have to first build it, and then run darwin-rebuild from there.

> There is also a nix-darwin installer. It dosen't work with flakes though.

```sh
# builds the darwinConfiguration.
# after insalling nix-darwin, we won't need to specify extra-experimental-features anymore
nix build .#darwinConfigurations.YourHostName.system --extra-experimental-features "nix-command flakes"

# the plan is to now run this to install nix-darwin with our configuration
# ./result/sw/bin/darwin-rebuild switch --flake . # this will fail as we first have to do the following lines

printf 'run\tprivate/var/run\n' | sudo tee -a /etc/synthetic.conf # read below
/System/Library/Filesystems/apfs.fs/Contents/Resouces/apfs.util -t # read below

# now we can finally darwin-rebuild
./result/sw/bin/darwin-rebuild switch --flake .  # install nix-darwin
```

> macOS dosen't allow any software to write to `/`.
> Instead you can write directory names or symlinks to `/etc/synthetic.conf`.
> 
> macOS will then create those files/symlinks on boot. (rebooting is boring, so we'll just run `apfs.util -t` to create them immediately)
>
>
> nix itself has just "nix" inside `/etc/synthetic.conf` (an empty folder at `/nix`), it'll then mount an apfs volume containing your nix store above it.
>
> nix-darwin needs a symlink from `/run` to `/private/var/run` to function, that's whats added in the printf|tee line

Your shell needs to source an rc file from nix-darwin to set up your environment variables. `/etc/static/zshrc` for zsh and `/etc/static/bashrc` for bash.

> that's what the `programs.zsh.enable` was for

so run

```bash
echo 'if test -e /etc/static/bashrc; then . /etc/static/bashrc; fi' | sudo tee -a /etc/bashrc
```

on bash and

```zsh
echo 'if test -e /etc/static/zshrc; then . /etc/static/zshrc; fi' | sudo tee -a /etc/zshrc
```

on zsh.
You may have to rerun this on macOS updates.

<br>

If you make changes to your configuration, you just have to commit them and run `darwin-rebuild switch --flake .` inside the repo.

<hr/>

I hope this post will guide some of you to the [god complex](https://www.reddit.com/r/NixOS/comments/kauf1m/dealing_with_post_nixflake_god_complex/).

When I find the time, I'll try guiding you to manage multiple systems in a single flake.

<br>

You can find all the code of this post [here](https://github.com/thexyno/blogpages/tree/main/nix-darwin-introduction).

And my NixOS/nix-darwin configurations [here](https://github.com/thexyno/nixos-config)
