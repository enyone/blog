# How to mount and decrypt LVM-luks encrypted hard disk

This helps you when you need to mount LVM-luks encrypted partition manually. This guide works at least in GNU/Linux Debian Jessie.

end of TL;DR so now go and read...

Finding correct device
-----------

Check what is the correct luks encrypted device. Change X in sdbX to whatever you disk and partition might be. If one is not luks encrypted command bellow will output:
```
$ sudo cryptsetup -v luksDump /dev/sdb1
Device /dev/sdb1 is not a valid LUKS device.
```

but if it is it will output something like bellow:
```
$ sudo cryptsetup -v luksDump /dev/sdbX
LUKS header information for /dev/sdbX
Version:       	1
Cipher name:   	aes
Cipher mode:   	xts-plain64
...
```

Opening the encryption
-----------

Use the passphrase you have used to store the key used to encrypt the partition. The key is stored into same partition and can be accessed using passphrase.
```
$ sudo cryptsetup luksOpen /dev/sdbX old-disk
Enter passphrase for /dev/sdbX: ****
```

Finding correct LVM volumes from inside encrypted partition
-----------

Usually you have created at least one logical LVM partition. In example bellow there is one logical volume. YYY is usually the hostname of the computer at where you encrypted the partition.
```
$ sudo lvdisplay 
  --- Logical volume ---
  LV Path                /dev/YYY-vg/root
  LV Name                root
  VG Name                YYY-vg
...
```

Activating LVM volumes
-----------

Volumes are non-active by default and you need to activate them before you are able to mount them.
```
$ sudo vgchange -a y YYY-vg
  1 logical volume(s) in volume group "YYY-vg" now active
```

Mounting
-----------

You can mount activated volumes just like normal partitions.
```
$ sudo udisks --mount /dev/YYY-vg/root
Mounted /org/freedesktop/UDisks/devices/dm_ZZZ at /media/whatever
```
