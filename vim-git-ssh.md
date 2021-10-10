# About

Methods to work on git repos housed on remote machines using vim, via SSH


## Method 1: sshfs

On your local machine, install sshfs:
```
sudo apt install sshfs
```

Then create a directory to house the remote machine and mount it:
```
mkdir -v LOCAL_MOUNT_POINT
sshfs REMOTE_SSH_CONFIG_NAME:/full/path/to/remote/repo  ./LOCAL_MOUNT_POINT -o idmap=user
```

changing the `LOCAL_MOUNT_POINT` and `REMOTE_SSH_CONFIG_NAME` appropriately.

Advantages:

- Works just as you would expect a normal filesystem. In particular, checking out git branches and other git based operations work
- Use local vim configuration and setup
- Almost no configuration required on the remote end

Disadvantages:

- For large file operations, dependent on network speed


## Default method: work on the remote machine directly

Advantages:

- No indirection. What you see is what you get

Disadvantages:

- Likely need to setup development environment on remote
- File editing operations subject to network speed; this may impact development experience depending on network connection


## References

- [https://unix.stackexchange.com/a/202919](https://unix.stackexchange.com/a/202919)
