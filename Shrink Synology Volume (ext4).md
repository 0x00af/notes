# How to shrink a Synology ext4 Volume:

I want to create a 20TB volume in a fully allocated Volume Group, so I have to shrink an existing Logical Volume by 20TB.

The [official way to do this](https://kb.synology.com/en-global/DSM/tutorial/Shrinking_existing_volume) is:
> You cannot decrease the size of an existing volume. Instead, you can remove the existing volume and create a new one. This article will guide you through the removal and creation process.

I don't accept this solution, so here we go.

***Disclaimer: This is not a bullet-proof way to shrink a volume. And *always* make sure you have a backup of the data you're messing with.***

- [1. Remove SSD Cache](#1-remove-ssd-cache)
- [2. Unmount Volume](#2-unmount-volume)
- [3. Check Volume Access](#3-check-volume-access)
- [4. Run a forced file system check](#4-run-a-forced-file-system-check)
- [5. Resize file system](#5-resize-file-system)
- [6. Shrink the Logical Volume](#6-shrink-the-logical-volume)
- [7. Extend file system to the new boundary](#7-extend-file-system-to-the-new-boundary)
- [8. Reboot](#8-reboot)
- [9. Do it in fewer steps](#9-do-it-in-fewer-steps)


## 1. Remove SSD cache

This is a precautionary measure. Maybe this is not necessary, but I didn't want to dive into that topic so I just removed the SSD cache from the volume.

## 2. Unmount Volume

When trying to unmount the volume, we will get a message that the volume is busy:

```
root@nas2:~# umount /volume1
umount: /volume1: target is busy.
```

Try to unmount it with `-l` lazy, `-f` force, and `-k` kill (undocumented):

```
umount -l -f -k /volume1
```

This should work. The NAS begins to beep, and in the GUI the volume is shown as `crashed`. That's fine for now.

## 3. Check Volume Access

Run `resize2fs -P` to see whether the volume is accessible (`-P` simply shows the estimated minimum space requirement for the volume).
We most probably get a message saying the volume is still busy:

```
root@nas2:~# resize2fs -P /dev/mapper/vg1-volume_1
resize2fs 1.44.1 (24-Mar-2018)
resize2fs: Device or resource busy while trying to open /dev/mapper/vg1-volume_1
Couldn't find valid filesystem superblock.
```

When that happens, we might need to remove a device mapper entry.

Check device mapper info:
```
root@nas2:~# dmsetup info
  
  Name:              vg1-volume_1
  State:             ACTIVE
  Read Ahead:        1536
  Tables present:    LIVE
  Open count:        1
  Event number:      0
  Major, minor:      248, 1
  Number of targets: 1
  UUID: LVM-Agh8dnThjdshl84hgYvcno8hhtgWnkoluibnfgAShjdnbGFghKBNM9d87fhjfZla

root@nas2:~# dmsetup info -c
Name                      Maj Min Stat Open Targ Event  UUID
vg1-volume_1              248   1 L--w    1    1      0 LVM-Agh8dnThjdshl84hgYvcno8hhtgWnkoluibnfgAShjdnbGFghKBNM9d87fhjfZla
```
Check `Open Count`. If it's not `0` `resize2fs` will not touch it.

Take note of the Major and Minor number, here it's 248 and 1.

Next check the `holders` of the block device, in `/sys/dev/block/<major>:<minor>/holders/` (need to escape `:`):
```
root@nas2:~# ls -la /sys/dev/block/248\:1/holders/
total 0
drwxr-xr-x 2 root root 0 Mar 11 21:56 .
drwxr-xr-x 9 root root 0 Mar 11 20:38 ..
lrwxrwxrwx 1 root root 0 Mar 11 22:01 dm-2 -> ../../dm-2
```

The device mapper entry `dm-2` marks the block device `248:1` as `open`. So, remove the mapping:
```
root@nas2:~# dmsetup remove /dev/dm-2
root@nas2:~# ls -la /sys/dev/block/248\:1/holders/
total 0
drwxr-xr-x 2 root root 0 Mar 11 21:56 .
drwxr-xr-x 9 root root 0 Mar 11 20:38 ..
```

Let's try `resize2fs -P` again. The given size is not bytes, but *blocks*. We use `tune2fs` to get the block size:
```
root@nas2:~# resize2fs -P /dev/mapper/vg1-volume_1 
resize2fs 1.44.1 (24-Mar-2018)
Estimated minimum size of the filesystem: 512646817

root@nas2:~# tune2fs -l /dev/mapper/vg1-volume_1 | grep 'Block size'
Block size:               4096
```

This volume is *estimated* to need `512646817` `4k` blocks, so ~1955GB of data. This number is not exact, but we only wanted to check device access. The next step will give us an exact minimum block count.


## 4. Run a forced file system check

You ***must*** run an `e2fschk -f` on the volume, and have to say ***yes*** to every proposed fix. resize2fs will refuse to do anything unless you say `yes` to the most basic "create a missing `lost+found` folder" prompt.
You could use `resize2fs -f` to ignore certain checks, but let's go ahead with the safer way.

Our volume1 with an estimated 1955GB data and a few files:
```
root@nas2:~# e2fsck -f /dev/mapper/vg1-volume_1 
e2fsck 1.44.1 (24-Mar-2018)
Pass 1: Checking inodes, blocks, and sizes
Inode 1143932981 extent tree (at level 1) could be shorter.  Fix<y>? yes
Inode 1337393160 extent tree (at level 1) could be narrower.  Fix<y>? yes
Inode 1337393164 extent tree (at level 2) could be narrower.  Fix<y>? yes
Pass 1E: Optimizing extent trees
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
/lost+found not found.  Create<y>? no
Pass 4: Checking reference counts

Pass 5: Checking group summary information

1.44.1-69057: ***** FILE SYSTEM WAS MODIFIED *****

1.44.1-69057: ********** WARNING: Filesystem still has errors **********

1.44.1-69057: 555924/1830092800 files (1.2% non-contiguous), 624455615/29281484800 blocks
```
`e2fsck` gives more accurate infos, there are `624455615` `4k` blocks in use (~2.382GB) of a total of `29281484800` blocks (109TB).

I said *no* to the prompt about `/lost+found`, because reasons.

`resize2fs` has a different opinion about the missing `/lost+found` folder:
```
root@nas2:~# resize2fs /dev/mapper/vg1-volume_1 10240G
resize2fs 1.44.1 (24-Mar-2018)
Please run 'e2fsck -f /dev/mapper/vg1-volume_1' first.

```

Since every prompt has to be yes anyway, let's just go ahead and run `e2fsck -fy`:

```
root@nas2:~# e2fsck -fy /dev/mapper/vg1-volume_1 
e2fsck 1.44.1 (24-Mar-2018)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
/lost+found not found.  Create? yes

Pass 4: Checking reference counts
Pass 5: Checking group summary information
1.44.1-69057: 555925/1830092800 files (1.2% non-contiguous), 624455616/29281484800 blocks
```

`e2fsck` is happy now.

Let's go ahead with `resize2fs`.

## 5. Resize file system

I want to shrink the volume by 20TB, so target would be around 89TB. For fun I resized the filesystem to 10TB. 

Depending on the data distribution this might take some time. The more "under" you go, the more "moving overhead" you get.
This might take a few hours.
```
root@nas2:~# resize2fs -p /dev/mapper/vg1-volume_1 10240G
resize2fs 1.44.1 (24-Mar-2018)
Resizing the filesystem on /dev/mapper/vg1-volume_1 to 2684354560 (4k) blocks.
Begin pass 2 (max = 6860546)
Relocating blocks             XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 3 (max = 893600)
Scanning inode table          XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 4 (max = 22709)
Updating inode references     XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
The filesystem on /dev/mapper/vg1-volume_1 is now 2684354560 (4k) blocks long.
```

### *! Only continue when the filesystem shrinking was successful !*

When you accidentally shrink the LV when the `resize2fs` was not successful, the data in the removed extents is lost.
You might get lucky when your volume was allocated from a continuous batch of PV extents, simply try to extend the LV again.

## 6. Shrink the Logical Volume

### *! Only do this **AFTER** successful file system adjustment, otherwise your file system is gone !*

Shrink the LVM Logical Volume:
```
root@nas2:~# lvm lvreduce -L -20T vg1/volume_1
WARNING: Reducing active logical volume to 89.08 TiB
THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce volume_1? [y/n]: y
Size of logical volume vg1/volume_1 changed from 109.08 TiB (28595359 extents) to 89.08 TiB (23352479 extents).
Logical volume volume_1 successfully resized.
```

## 7. Extend file system to the new boundary

Now let's grow the file system the new boundaries (might take hours again):

```
root@nas2:~# resize2fs /dev/mapper/vg1-volume_1
resize2fs 1.44.1 (24-Mar-2018)
Resizing the filesystem on /dev/mapper/vg1-volume_1 to 23912938496 (4k) blocks.
The filesystem on /dev/mapper/vg1-volume_1 is now 23912938496 (4k) blocks long.
```

## 8. Reboot

I didn't care to re-create the device mapper entries, they're runtime only anyways. Reboot the NAS, and you're set.

After a reboot, the Major:Minor number is now 249:1
```
Name                      Maj Min Stat Open Targ Event  UUID                                                              
vg1-volume_1              249   1 L--w    1    1      0 LVM-Agh8dnThjdshl84hgYvcno8hhtgWnkoluibnfgAShjdnbGFghKBNM9d87fhjfZla
```

Also, after the reboot, the NAS is now somehow busy with `Optimizing file system... 12.83%`, just like it was after the initial volume creation. I assume that's some kind of proprietary index.


## 9. Do it in fewer steps

All numbers are based on a rather rough/lazy calculation. When you got your numberes right, you can do it in fewer steps.

Example:
* 4KiB (4096B) file system blocks (check `tune2fs -l /dev/mapper/vg1-volume_1 | grep "Block size"`)
* 4MiB (4194304B) LVM PE size (check `vgdisplay vg1 | grep "PE Size"`)
* Target: 15TB (3932160 PV extents, 4026531840 file system blocks)
```
root@nas2:~# resize2fs -p /dev/mapper/vg1-volume_1 4026531840
...
root@nas2:~# lvm lvreduce -l 3932160 vg1/volume_1
...
```
