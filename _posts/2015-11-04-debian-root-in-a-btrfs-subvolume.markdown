---
layout:     post
title: 		"Debian root in a btrfs subvolume"
date: 		2015-11-04 10:09:49 -0500
categories: debian installation
---

# Background

If you select btrfs as your filesystem of choice in the Ubuntu 14.04 installer,
it'll create [btrfs
subvolumes](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes)
for any "partitions" you request. These partitions will be subvolumes named `@`
for `/`, `@home` for `/home`, etc.

Sadly, while the Debian installer will install into btrfs, it will not use
subvolumes. Thus, after installing Debian, if you want to get your root
filesystem into a subvolume, you have to do a bit of work. I wanted to do this
because it makes things like snapshots much cleaner when the OS is not
installed directly in the btrfs root (otherwise you can't have any space in
btrfs "outside" the mounted filesystem).

# Process

1. Install Debian as normal.

2. Once installed, I like to immediately create a snapshot of the installed
   system:

```
# btrfs subvol snap -r / /@stretch_fcs
```

3. Create the snapshot that will become the new root fs:

```
# btrfs subvol snap /@stretch_fcs /@stretch
```

4. Update grub config in the snapshot to refer to the subvolume ([thanks
   goncalopp@stackexchange](http://unix.stackexchange.com/questions/62802/move-a-linux-instalation-using-btrfs-on-the-default-subvolume-subvolid-0-to-an)).
   My procedure is slightly different since I'm using EFI boot (mounting
   `/boot/efi` in the chroot is probably not even necessary, but, just in
   case). If your /boot is a separate partition, you would still need to mount
   `/boot` before `/boot/efi`:

```
# btrfs mount -o subvol=@stretch /mnt
# cd /mnt
# mount -o bind /dev dev
# mount -o bind /sys sys
# mount -o bind /proc proc
# mount -o bind /boot/efi boot/efi
# chroot .
# update-grub
# exit
```

5. Update the subvolume's `/etc/fstab` to mount the subvolume as the root
   filesystem. Excuse the lazy regexes, you probably want to just edit
   `/etc/fstab` by hand, but I don't like putting "edit this file" in
   instructions):

```
# grep '^[^#].* / ' /etc/fstab
UUID=71d1dc27-36d5-492b-b51c-f852fdaa6b4e /               btrfs   defaults        0       1
# sed -ie '/ \/ /s#defaults\([^ ]*\) *#defaults\1,subvol=@stretch #' /tmp/fstab
# grep '^[^#].* / ' /etc/fstab
UUID=71d1dc27-36d5-492b-b51c-f852fdaa6b4e /               btrfs   defaults,subvol=@stretch 0       1
```

6. Set the default btrfs filesystem to the `@stretch` snapshot. We will undo
   this later because leaving the subvolume as the default means that it's more
   cumbersome to mount the btrfs root. However, if we were to reboot now
   without changing the default, Grub would boot using `/boot/grub/grub.cfg`
   rather than `/@stretch/boot/grub/grub.cfg`, and `/` would be the mounted
   filesystem once booted. (Aside, `btrfs` can't select a subvolume by name and
   just print its id? Annoying):

```
# btrfs subvol set-default $(btrfs subvol list / | awk '$NF == "@stretch" { print $2 }') /
```

7. Reboot:

```
# reboot
```

8. Verify that you have booted with `@stretch` as the root filesystem:

```
/
	Name: 			@stretch
	uuid: 			284d7ea5-f406-2541-953f-60772fc247af
	Parent uuid: 		09184ee6-f5f9-5a4c-9d1b-2557b0541f0a
	Creation time: 		2015-11-04 10:04:10
	Object ID: 		282
	Generation (Gen): 	502
	Gen at creation: 	482
	Parent: 		5
	Top Level: 		5
	Flags: 			-
	Snapshot(s):
```

9. Reset `/` as the default subvolume (the btrfs root's subvolume id is always 5):

```
# btrfs subvol set-default 5 /
```

10. Reinstall grub so that it knows to read its config from
    `/@stretch/boot/grub/grub.cfg` (assuming your boot device is `/dev/sda`):

```
# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.2.0-1-amd64
Found initrd image: /boot/initrd.img-4.2.0-1-amd64
done
# update-grub /dev/sda
Installing for x86_64-efi platform.
Installation finished. No error reported.
```

11. Reboot:

```
# reboot
```

12. You should now be able to mount `/dev/sda3` (or whatever your btrfs device
    is) and see the entire tree. I just put it in /etc/fstab and keep it
    mounted at /btrfs:

```
# grep /btrfs /etc/fstab
UUID=71d1dc27-36d5-492b-b51c-f852fdaa6b4e /btrfs          btrfs   defaults      0       1
# mkdir /btrfs && mount /btrfs
# ls -l /btrfs
bin   dev  home        lib    media  opt   root  sbin  @stretch      sys  usr  vmlinuz
boot  etc  initrd.img  lib64  mnt    proc  run   srv   @stretch_fcs  tmp  var
```

13. If you want, you can now remove `<btrfs_root>/<non-subvol-contents>` e.g.:

```
# cd /btrfs
# rm -rf bin boot dev etc home initrd.img lib lib64 media mnt opt proc root run sbin srv sys tmp usr var vmlinuz
```

Now `/btrfs` is free for you to create things in, e.g. a `snapshots/` directory
in which you can create your btrfs snapshots.

# Conclusion


# TODO

My desktop originally ran Ubuntu 14.04. I later wanted to switch to Debian
Stretch without wiping out Ubuntu. I can now dual boot between them thanks to
btrfs subvolumes and UEFI. So I need to document that...
