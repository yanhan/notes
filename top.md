# About

Notes on the `top` utility. This is for the Linux version; the Mac version is slightly different.


## Stuff you see at the top of the screen

### Load average values

The load average values are located at the top right corner of the screen. They look like the following:

```
load average: 0.45, 0.57, 0.62
```

These 3 numbers are the 1 min, 5 min and 15 min load average values respectively.

**Simple way to interpret load averages:** If the load average is 1.00 and the CPU has 1 core, the server is at capacity. With 2 cores, server is at capacity when the number is 2.00. With 4 cores, this number should be 4.00. And so on.

**Longer explanation:** Think of a CPU core as a road and a process as a car. If there is always 1 car on the road, the load average is 1.00. If there are 2 cars, then the load average is 2.00 and 1 car can be on the road while the other car has to wait for the road to be free. Hence load average is **very roughly** `number of process that need to run / number of CPU cores` and measures how overloaded a server is.

**A simple rule of thumb:** If the 15 min load average exceeds 0.7 (after dividing by the number of CPU cores), then the server may be overloaded.

### CPU percentage numbers

- user time `(us)`
- system time `(sys)`
- time spent on low priority processes aka nice time `(ni)`
- time spent in wait for I/O processes `(wa)`
- time handling hardware interruptions `(hi)`
- time handling software interruptions `(si)`
- time stolen from virtual machine `(st)`


## Columns

- `PR`: task's priority. From -20 to 19, with -20 being most important
- `NI`: nice value, which augments priority of task. Negative number increases task's priority, positive number decreases it
- `VIRT`: virtual memory used (combo of RAM and swap)
- `RES`: resident size of non-swapped, physical memory in KBs
- `SHR`: shared memory size, memory that can be allocated to other processes
- `S`: process status. Can be running `(R)`, sleeping and unable to be interrupted `(D)`, sleeping and able to be interrupted `(S)`, trace / stopped `(T)`, zombie `(Z)`
- `TIME+`: cumulative CPU time that the process and children processes have used


## Interactive commands

- `M`: sort by memory usage
- `P`: sort by CPU usage
- `s`: change refresh time (will be prompted to enter a value)
- `Space / Enter`: refresh
- `n`: change number of processes shown (will be prompted to enter a value)
- `k`: kill process (will be prompted to enter a value for the PID)
- `f`: see list of fields and you can choose which to display. Use up and down keys to navigate, press `d` to toggle display, press `s` to select as sort field
- `H`: show individual threads for all processes
- `i`: toggle whether idle processes are shown
- `U / u`: filter by username
- `1`: toggle between all CPUs as a whole vs. CPU by core
- `L`: locate string
- `w`: write config file
- `h`: open help


## Command line options

- `-n 10`: shows `10` iterations of information and then quit
- `-b`: batch mode: just prints information on processes every specified number of seconds until all iterations run out (specified with `-n`)
- `-d[interval]`: set delay time that top uses to refresh results
- `-i`: toggle whether idle processes are shown
- `p[PID,PID]`: filter to only show the specified processes
- `-u [username]`: filters by user


## References

- http://www.tech-faq.com/how-to-use-the-unix-top-command.html
- https://www.linode.com/docs/uptime/monitoring/top-htop-iotop/
- https://coskan.wordpress.com/2008/12/22/how-to-use-top-effectivelly-on-linux-as-a-dba/
