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
- When building / testing, need to remember to do it on the remote instead of locally


## Method 2: Push to git remote on the server

On your local machine, add the remote git repo as a git remote:
```
git remote add REMOTE_NAME  REMOTE_SSH_CONFIG_NAME:/full/path/to/remote/repo
```

Then work locally, and push to the remote when changes are made:
```
git push REMOTE_NAME BRANCH_NAME
```

If the remote repo is not a bare repo and the branch is checked out and you still want to push to it, on the remote repo, run:
```
git config receive.denyCurrentBranch warn
```

Advantages:

- Use local filesystem
- Use local development environment
- Almost no configuration required on the remote end
- No need to install any additional software. Only requires SSH and existing core git feature

Disadvantages:

- If the remote is not a bare repo, by default, cannot checkout the branch you will be pushing (but see above on how to allow this)
- If remote is not bare and the branch is checked out, may need to run `git reset --hard HEAD` to see latest changes reflected
- When building / testing, need to remember to do it on the remote instead of locally


## Default method: work on the remote machine directly

Advantages:

- No indirection. What you see is what you get

Disadvantages:

- Likely need to setup development environment on remote
- File editing operations subject to network speed; this may impact development experience depending on network connection


## References

- [https://unix.stackexchange.com/a/202919](https://unix.stackexchange.com/a/202919)
