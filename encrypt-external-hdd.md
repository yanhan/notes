# About

How to encrypt external hard disk

Assumption: The hard disk is `/dev/sdb`


## Stronger encryption preparation (optional)

**NOTE:** This can take hours to complete.

One of the following 3 commands (ordered from shortest to longest time to complete):

```
sudo dd if=/dev/zero of=/dev/sdb bs=4K
sudo badblocks -c 10240 -s -w -t random -v /dev/sdb
sudo dd if=/dev/urandom of=/dev/sdb bs=4K
```

First command writes zeros to the hard disk.

Second command performs badblock scan of hard disk to detect an early failure, while writing random data to it at the same time. It also provides visibility of progress.

Third command is most secure but will take a looong time.


## Create partition table

Run:

```
sudo fdisk /dev/sdb
```

Then create a partition table and write it to disk.


## Check for kernel modules

If these exist, the command should return zero exit code and does not output anything:

```
sudo modprobe dm-crypt
sudo modprobe sha256
sudo modprobe aes
```

If the `sudo modprobe aes` fails, try:

```
sudo modprob aes_generic
```

Then

```
alias aes aes_generic
```


## Encrypt the partition

This encrypts `/dev/sdb1`. Change it accordingly:

```
sudo cryptsetup --verify-passphrase luksFormat /dev/sdb1 -c aes -s 256 -h sha256
```


## Create filesystem

This unlocks the encrypted partition and maps it to `/dev/mapper/securebackup`. Feel free to change the `securebackup` to something else:

```
sudo cryptsetup luksOpen /dev/sdb1 securebackup
```

Then format the partition:

```
sudo mkfs.ext4 -m 1 -O dir_index,filetype,sparse_super /dev/mapper/securebackup
```

Explanation of the options:

- `-m 1`: reduce filesystem blocks reserved for superuser from 5% to 1%. Useful for large filesystems
- `-O dir_index`: speeds up lookups in large filesystems
- `-O filetype`: stores filetype info in directories
- `-O sparse_super`: create fewer superblock backup copies - useful for large filesystems


## To use this

Just umount the hard disk then plug it in again. If using a GUI, it should be recognized and you will be prompted for the passphrase.

You might need to chown the mount point in order to write to the hard disk.


## References

- https://help.ubuntu.com/community/EncryptedFilesystemsOnRemovableStorage
