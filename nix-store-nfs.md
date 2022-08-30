---
id: nix-store-nfs
title: Mounting a nix-store via NFS
created: 2022-08-22
updated: 2022-08-22
tags:
  - nix
  - NixOS
  - nfs
---

Lately at $work we wanted to netboot some (NixOS) Computers.
Having a full nix-store for each one on some Server seemed wasteful though, so I searched for another solution.

NixOS does not need a full rootfs to boot, it just needs `/nix` (and the pseudo file systems like `/dev`, `/proc`,...).
All the other directories get generated inside the activation script.

<tangent>
That's how [Impermanence](https://github.com/nix-community/impermanence) (and the NixOS iso) works
</tangent>

The path seemed clear:

1. Netboot the kernel/initramfs from HTTP/TFTP
2. Mount the nix-store via NFS inside the initramfs
3. Profit

Multiple instances of nix writing onto the same store might mean problems.
So we'll just mount the nix store read only.

| But we might want to write to it

So we'll overlay a rw directory over the read only store.

I just naively tried

```nix
{ ... }: {
  boot.initrd.supportedFilesystems = [ "nfs" "nfsv4" "overlay" ];   # load needed kernel modules
  boot.initrd.availableKernelModules = [ "nfs" "nfsv4" "overlay" ]; # load them again, because of cause it didn't work
  fileSystems."/" = { device = "tmpfs"; fsType = "tmpfs"; options = [ "size=2G" ]; };
  fileSystems."/nix/.rw-store" = { fsType = "tmpfs"; options = [ "mode=0755" "size=8G" ]; neededForBoot = true; };
  fileSystems."/nix/.ro-store" =
    {
      fsType = "nfs4";
      device = "192.168.1.1:nixstore";
      options = [ "ro" ];
      neededForBoot = true;
    };
  fileSystems."/nix/store" =
    {
      fsType = "overlay";
      device = "overlay";
      options = [
        "lowerdir=/nix/.ro-store"
        "upperdir=/nix/.rw-store/store"
        "workdir=/nix/.rw-store/work"
      ];
      depends = [
        "/nix/.ro-store"
        "/nix/.rw-store/store"
        "/nix/.rw-store/work"
      ];
    };
  boot.initrd.network.enable = true;
  networking.useDHCP = true;
}
```

but it just didn't boot.
The initramfs ran through, but then it just hung.

<tangent>
> `boot.shell_on_fail` and the [other cmdline options](https://github.com/NixOS/nixpkgs/blob/e2f8343087e0b131562a7c1fef220abccb6d1981/nixos/modules/system/boot/stage-1-init.sh#L185) really helped debugging
</tangent>

NixOS initramfs networking yeets the network configuration after booting successfully.

<tangent>
> it took me at least a solid 10h to find that out
</tangent>

You have to set `boot.initrd.network.flushBeforeStage2` to `false` to disable that.

And now it just works™


Just for completeness, a full configuration is:

```nix
{ ... }: {
  boot.initrd.supportedFilesystems = [ "nfs" "nfsv4" "overlay" ];   # load needed kernel modules
  boot.initrd.availableKernelModules = [ "nfs" "nfsv4" "overlay" ]; # load them again, because of cause it didn't work
  fileSystems."/" = { device = "tmpfs"; fsType = "tmpfs"; options = [ "size=2G" ]; };
  fileSystems."/nix/.rw-store" = { fsType = "tmpfs"; options = [ "mode=0755" "size=8G" ]; neededForBoot = true; };
  fileSystems."/nix/.ro-store" =
    {
      fsType = "nfs4";
      device = "192.168.1.1:nixstore";
      options = [ "ro" ];
      neededForBoot = true;
    };
  fileSystems."/nix/store" =
    {
      fsType = "overlay";
      device = "overlay";
      options = [
        "lowerdir=/nix/.ro-store"
        "upperdir=/nix/.rw-store/store"
        "workdir=/nix/.rw-store/work"
      ];
      depends = [
        "/nix/.ro-store"
        "/nix/.rw-store/store"
        "/nix/.rw-store/work"
      ];
    };
  boot.initrd.network.enable = true;
  boot.initrd.network.flushBeforeStage2 = false; # otherwise nfs dosen't work
  networking.useDHCP = true;
}
```

The netboot stuff we built is open source and available [here](https://github.com/nix-basement/nix-basement/), but not ready for production yet.
