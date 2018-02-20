# About

My notes on Bash scripting.


## Finding files / directories

### locate

- The `locate` program is from GNU findutils. It quickly finds filenames that match shell-like wildcard patterns from a compressed database of names. Wildcards must be quoted to prevent shell expansion
- This compressed database of names is created by `updatedb`
- **NOTE:** To avoid privacy issues, run `updatedb` as a non-privileged users so all the files in the database are those that can be found by any user. The alternative is to use `slocate`


## String matching

- `${var#pattern}` - if `pattern` matches **start** of `var`'s value, delete **shortest** match and return the rest
- `${var##pattern}` - same as above but delete **longest** match
- `${var%pattern}` - if `pattern` matches **end** of `var`'s value, delete **shortest** match and return the rest
- `${var%%pattern}` - same as above but delete **longest** match

How to remember?

- 2 signs (`##` and `%%`) = longest match; 1 sign (`#` and `%`) = shortest match
- `#` matches start because it is like a comment; `%` comes after numbers so it matches end


## Files and directories

### Show unprintable characters in filenames

Use the `od` program to show unprintable characters say in filenames:

For instance:

```
ls one*two | od -a -b
```

could output:

|000000|o|n|e|nl|t|w|o|nl|
|:---|:---|:---|:---|:---|:---|:---|:---|:---|
||157|156|145|012|164|167|157|012|
|000010|||||||||

That revealed 2 newline characters (the `nl`s) in the filename

### Create temporary file / directory securely

Use `mktemp`

- It creates a file with **no access** for group and others
- For the template string (the `XXX...` part), it seems that the process ID is included. Which makes the filename more guessable. Advice is to use a longer template string.


## Shell options

- `set -f`: disables wildcard expansion
- `set -n`: read commands and check for syntax errors but don't execute them

### Get enabled shell options

`$-` is a string representing the currently enabled shell options


## Functions

### Remove function

```
unset -f func_name
```


## Environment variables

### Ignore inherited environment for child processes

Use the `env` command with the `-i` option to ignore the inherited environment when spinning up a child process and only use those variables specified on the command line.

```
env -i PATH=$PATH HOME=$HOME LC_ALL=C  awk '...' file1 file2
```

The `awk` process will only have the `PATH`, `HOME` and `LC_ALL` environment variables set.


## sed

### Replace nth occurrence of string

You can specify a trailing number `n` to indicate that the nth ocurrence should be replaced.

```
sed 's/Tolstoy/Camus/2' <<< "Tolstoy reads well. Tolstoy writes well."
```

Output is:

```
Tolstory reads well. Camus writes well.
```


## Generating randomness

- `/dev/random`: will block until sufficient randomness has been gathered to guarantee high quality random data
- `/dev/urandom`: never blocks but data is less random


## Security

### Prevent spoofing attacks

Hashbang line which prevents spoofing attacks:

```
#!/bin/sh -
```

The dash character says that there are no more shell options.

#### Longer explanation

Suppose there's a setuid script named `/etc/setuid_script` which begins with:

```
#!/bin/sh
```

If you run the following command:

```
cd /tmp
ln /etc/setuid_script -i
PATH=.
-i
```

The final command will be rearranged to

```
/bin/sh -i
```

which gives an interactive shell that's setuid to the owner of the script.

This can be prevented by making the 1st line:

```
#!/bin/sh -
```


### Secure shell script tips

- every directory on `PATH` environment variable should only be writable by its owner. Same for every program in those directories
- Don't trust passed in environment variables. Check and reset them if they will be used by subsequent commands. eg. `TZ`, `PATH`, `IFS`. Good practice: explicitly set `PATH` to contain only system bin directories and explicitly set `IFS`
- `cd` to a known directory so subsequent relative pathnames are in known location. Be sure the `cd` succeeds. This can be done using `cd app-dir || exit 1`
- Use syslog / similar to create an audit trail. Log the date and time of invocation, username, etc
- Always quote user input when using it to prevent malicious input from being further evaluated
- Quote results of wildcard expansion
- Check user input for metacharacters, eg. $, \`
- Be aware of race conditions. If an attacker can execute arbitrary commands between any 2 commands, will it compromise security?
- Check if a file is a symbolic llink (use `[ -L file ]` or `[ -h file ]`) to make sure it is not a symlink to critical system file before editing / `chmod`ing it
- If necessary, use setgid instead of setuid
- Use dedicated non-root user if setuid is necessary
- Limit setuid code as much as possible. Move it to a separate program in fact. Code as if it can be invoked by anyone!
- There are mount options to disable setuid / setgid bit for entire filesystems. It is a good idea for network-mounted filesystems, CD-ROMS, etc.

### Prelude recommended by Charles Ramey, maintainer of bash

```
IFS=$' \t\n'

# Make sure unalias isn't a function, since it is a regular built-in.
# unset is a special built-in, sot it will be found before functions
unset -f unalias

# unset all aliases and quote unalias (using the backslash) so it is not
# alias-expanded
\unalias -a

# Make sure command isn't a function, since it's a regular built-in.
# unet is a special built-in, so it will be found before functions.
unset -f command

# Get a reliable PATH prefix, handling case where getconf is not available
SYSPATH="$(command -p getconf PATH 2>/dev/null)"

if [[ -z "$SYSPATH" ]]; then
  SYSPATH="/usr/bin:/bin" # pick your poison
fi

PATH="$SYSPATH:$PATH"
```


## References

- [Classic Shell Scripting](http://a.co/0w5eKr4)
