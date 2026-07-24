# Defeating ftrace based LKM rootkits

Let's get right to it!

Defeating ftrace based rootkits IS possible. Here, I will show you a few things that might be of help to you in identifying and disabling ftrace based rootkits..
First, we are going to look at kernel tracing. Very simple to do, yet a very powerful tool!<br><br>

```
sudo cat /sys/kernel/tracing/enabled_functions
```
Below we can see an example of what this can tell us.

```
__x64_sys_kill (1) R I     M 	tramp: 0xffffffffc05dc000 (fh_ftrace_thunk+0x0/0xa0 [deception]) ->ftrace_ops_assist_func+0x0/0x100
__x64_sys_getdents64 (1) R I     M 	tramp: 0xffffffffc05ce000 (fh_ftrace_thunk+0x0/0xa0 [deception]) ->ftrace_ops_assist_func+0x0/0x100
__x64_sys_recvmsg (1) R I     M 	tramp: 0xffffffffc05c9000 (fh_ftrace_thunk+0x0/0xa0 [deception]) ->ftrace_ops_assist_func+0x0/0x100
tcp6_seq_show (1) R I     M 	tramp: 0xffffffffc05dd000 (fh_ftrace_thunk+0x0/0xa0 [deception]) ->ftrace_ops_assist_func+0x0/0x100
```

We can see exactly what is being hooked as well as the name of the kernel module doing the hooking! This shows us the module [deception] is at work here, even though the module is hidden.
This is extremely helpful. Here the module was inserted without its name being changed. So we now know that the rootkit deception is installed on our system.
If you want to further inspect what the module is doing, you can temporarily disable ftrace in order to see what the module is doing. I.E, hiding ports or processes.
```
sudo sysctl kernel.ftrace_enabled=0
```
Once you run this command, you can use system tools like ss and netstat and ps to show you what the rootkit has been hiding from you and what it is currently doing.<br>

Since we now know the name of the rootkit, we can look at options to completely disable and remove it. A lot of rootkits have simple kill commands to control the kernel module such as kill -63 0
to make module visible in lsmod or kill -64 0 to give root privileges. We can simply use
```
kill -63 0
```
to unhide the module, followed by
```
sudo rmmod deception
```
However, in the case of deception, the exact command is variable according to a user defined PID (or magic number). For example, the exact command to reveal the module might be kill -63 8000. So we don't know in this case
what the exact command is. So now what? Let's assume the module has not been made persistent by the use of a cronjob, dkms or a service file. Also, /etc/modules-load.d/ (variable on different distros) can load modules while
booting. A kernel module, once inserted, if not made persistent, can be cleared from the kernel simply by rebooting. If a reboot does not resolve the problem, it might be being made persistent by one of these methods.
So lets start with cronjobs. We can check our cronjobs like this
```
crontab -l
```
Be sure to check roots crontab as well as your own user. If a suspicious entry is present, you can stop it from executing by placing a # at the beginning of the line.<br><br>
Now lets move on to dkms
```
sudo dkms status
```
This will show you any modules that are being loaded by dkms during boot. If you find a suspicious module, you can remove it by using
```
sudo dkms remove deception/1.0
```
Another place, that can start a module during boot is service files, which reside in /etc/systemd/system on systemd based linux distros. So take a look here and look through the service files. If you find a suspicious one,
you can use the following
```
sudo systemctl stop deception
```
and also,
```
sudo systemctl disable deception
```
This will prevent the module from loading during boot.<br><br>
Finally, the last place we want to look is the /etc/modules-load.d/ directory. The exact directory can be slightly different depending on your distro. Here you might find a number of files named *.conf and will contain
the line (for example)
```
deception
```
You can disable it like this
```
sudo mv /etc/modules-load.d/filename.conf /etc/modules-load.d/filename.conf.disabled
```

Once you have identified the modules means of staying persistent and have followed these steps, simply reboot and your system should be free of the rootkit!<br>
### What about LKM rootkits that don't use ftrace?
In the case of diamorphine rootkit or other rootkits which use kprobes. You can identify the hidden module as well as a list of available kernel functions using
```
sudo cat /sys/kernel/tracing/available_filter_functions
```
And the result in the case of diamorphine
```
hacked_getdents64 [diamorphine]
hacked_getdents [diamorphine]
resolve_sym [diamorphine]
get_syscall_table_bf [diamorphine]
find_task [diamorphine]
is_invisible [diamorphine]
give_root [diamorphine]
module_show [diamorphine]
module_hide [diamorphine]
hacked_kill [diamorphine]
flipswitch_func [diamorphine]
```
Diamorphine LKM as we know is revealed in lsmod with
```
kill -63 0
```
Once revealed, you can remove it using
```
sudo rmmod diamorphine
```



