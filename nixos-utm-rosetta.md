---
id: nixos-utm-rosetta
title: Using Rosetta 2 in a NixOS VM
created: 2022-09-07
updated: 2023-07-28
tags:
  - nix
  - NixOS
  - macOS
  - UTM
---

<box>
Update (2023-02-09): [There is a bug in Rosetta right now preventing to run some GUI applications](https://github.com/NixOS/nixpkgs/issues/209242)

Update (2022-10-29): Removed parts about how everything is in beta, as macOS Ventura and UTM 4 are out of beta.


This only works with macOS 13 (or later) and UTM 4 (or later)
</box>

With the release of macOS Ventura, Apple *generously* allows people to also use Rosetta 2 in their Linux VMs.

<tangent>
It of course only works in Linux VMs on Apple Silicon macs, [althrough seemingly you could just take the binary and put it onto other (ARM) computers](https://twitter.com/never_released/status/1534127641082593281)
</tangent>


To use it, you have to use a VM Software utilizing [Virtualization.Framework](https://developer.apple.com/documentation/virtualization) (not to be confused with [Hypervisor.Framework](https://developer.apple.com/documentation/hypervisor)), which also exposes the option to mount the Rosetta volume, and you have to be on macOS 13 (Beta).

We'll use [UTM](https://mac.getutm.app/) here. 
The Rosetta option is only exposed on [UTM Version 4.0 and above](https://github.com/utmapp/UTM/releases) (you can also get it from the [Mac App Store](https://apps.apple.com/de/app/utm-virtuelle-maschinen/id1538878817)).

While creating the VM, check both "Use Apple Virtualization" and "Enable Rosetta"

Then install NixOS like normal (using an [aarch64 ISO](https://nixos.wiki/wiki/NixOS_on_ARM/UEFI#Getting_the_installer_image_.28ISO.29)).


To mount the Rosetta `virtiofs`, register the binary format and tell nix that you can build x86 now, put the following inside your NixOS configuration:

<greybox>
Update (2023-07-28): On current NixOS (23.05) you can just do `virtualisation.rosetta.enable = true;` instead of the following
</greybox>


```nix
{ config, lib, ...}: {
  boot.initrd.availableKernelModules = [ "virtiofs" ];
  fileSystems."/run/rosetta" = {
    device = "rosetta";
    fsType = "virtiofs";
  };
  nix.settings.extra-platforms = [ "x86_64-linux" ];
  nix.settings.extra-sandbox-paths = [ "/run/rosetta" "/run/binfmt" ];
  boot.binfmt.registrations."rosetta" = { # based on https://developer.apple.com/documentation/virtualization/running_intel_binaries_in_linux_vms_with_rosetta#3978495
    interpreter = "/run/rosetta/rosetta";
    fixBinary = true;
    wrapInterpreterInShell = false;
    matchCredentials = true;
    magicOrExtension = ''\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00'';
    mask = ''\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff'';
  };
}
```

`nixos-rebuild`

And now you can run X86 Applications

```sh
$ nix-shell -E 'with import <nixpkgs> { system="x86_64-linux"; }; runCommand "dummy" { buildInputs = [ hello ]; } ""' --command hello
Hello, world!
$
$ # or `nix run nixpkgs#legacyPackages.x86_64-linux.hello` if you're already using the new nix cli
```

<tangent>
Using nix-command is such a breeze compared to how it was before
</tangent>

Rosetta seems to work pretty good. In my testing (mostly building NixOS configuratinos to deploy to servers) everything just worked and UTM only crashed once.

Setting the VM up as a [remote builder](https://nixos.org/manual/nix/stable/advanced-topics/distributed-builds.html) is also really convenient.
