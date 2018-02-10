# About

Notes on the `top` utility. This is for the Linux version; the Mac version is slightly different.

## Top of the screen

Load average values: 1 min, 5 min, 15 min. If load average is 1.00 and CPU has 1 core, server is at capacity. With 2 cores, server is at capacity when the number is 2.00. With 4 cores, this number should be 4.00. And so on.

CPU percentage numbers:
- user time `(us)`
- system time `(sys)`
- time spent on low priority processes aka nice time `(ni)`
- time spent in wait for I/O processes `(wa)`
- time handling hardware interruptions `(hi)`
- time handling software interruptions `(si)`
- time stolen from virtual machine `(st)`


## Columns

`PR`: task's priority. From -20 to 19, with -20 being most important
`NI`: nice value, which augments priority of task. Negative number increases task's priority, positive number decreases it
`VIRT`: virtual memory used (combo of RAM and swap)
`RES`: resident size of non-swapped, physical memory in KBs
`SHR`: shared memory size, memory that can be allocated to other processes
`S`: process status. Can be running `(R)`, sleeping and unable to be interrupted `(D)`, sleeping and able to be interrupted `(S)`, trace / stopped `(T)`, zombie `(Z)`
`TIME+`: cumulative CPU time that the process and children processes have used


## Interactive commands

`M`: sort by memory usage
`P`: sort by CPU usage
`s`: change refresh time (will be prompted to enter a value)
`Space / Enter`: refresh
`n`: change number of processes shown (will be prompted to enter a value)
`k`: kill process (will be prompted to enter a value for the PID)
`f`: see list of fields and you can choose which to display. Use up and down keys to navigate, press `d` to toggle display, press `s` to select as sort field
`H`: show individual threads for all processes
`i`: toggle whether idle processes are shown
`U / u`: filter by username
`1`: toggle between all CPUs as a whole vs. CPU by core
`L`: locate string
`w`: write config file
`h`: open help


## Command line options

`-n 10`: shows `10` iterations of information and then quit
`-b`: batch mode: just prints information on processes every specified number of seconds until all iterations run out (specified with `-n`)
`-d[interval]`: set delay time that top uses to refresh results
`-i`: toggle whether idle processes are shown
`p[PID,PID]`: filter to only show the specified processes
`-u [username]`: filters by user


## References

- http://www.tech-faq.com/how-to-use-the-unix-top-command.html
- https://www.linode.com/docs/uptime/monitoring/top-htop-iotop/
- https://coskan.wordpress.com/2008/12/22/how-to-use-top-effectivelly-on-linux-as-a-dba/
