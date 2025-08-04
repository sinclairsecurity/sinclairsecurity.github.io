In my research with Linux security, I inevitably stumbled upon **capabilities**, which are ways to give an executable fine-grained and specific privileged permissions that it can leverage without requiring full administrator privileges. You can learn more about Linux capabilities here: ![Linux Capabilities](../tutorials/linux-capabilities.html)Linux Capabilities.

To aid me in my research, I wrote a utilitiy to help me perform quick analysis called `bitcap`. `bitcap` is a very simple, very fast way to quickly get the allowed capabilities out of a hexadecimal string. You will find capabilities reported as hexadecimal numbers quite often when exploring processes and their permissions. 

`bitcap` takes in a bitmask as hexadecimal and prints the capabilities associated with it. For instance:

```bash
chris@sinclairsecurity:~$ bitcap 0000000000001ff0
Capabilities:
CAP_FSETID
CAP_KILL
CAP_SETGID
CAP_SETUID
CAP_SETPCAP
CAP_LINUX_IMMUTABLE
CAP_NET_BIND_SERVICE
CAP_NET_BROADCAST
CAP_NET_ADMIN
```

Above, the hexadecimal number `0x0000000000001ff0` converts to 
```
0000000000000000000000000000000000000000000000000001111111110000
```
in binary, with bits 0 - 50 unset, bits 51 - 59 set, and bits 60-63 unset. This means the capabilities defined at places 51 through 59 are turned **on**, and the executable will have those permissions.

The capabilities of a process can normally be found by running `cat /proc/<PID>/status`, which are presented in hexadecimal.
We can chain together a few commands to find the permissions of any running process we would like:

```bash
chris@sinclairsecurity:~$ PROCESS=kthreadd; cat /proc/$(pidof ${PROCESS} | awk '{print $1}')/status | grep CapEff | awk '{print $2}' | xargs bitcap
Capabilities:
CAP_CHOWN
CAP_DAC_OVERRIDE
CAP_DAC_READ_SEARCH
CAP_FOWNER
CAP_FSETID
...
```

How is this done?
```c
const char *cap_names[] = {
    "CAP_CHOWN", "CAP_DAC_OVERRIDE", "CAP_DAC_READ_SEARCH",
    "CAP_FOWNER", "CAP_FSETID", "CAP_KILL", "CAP_SETGID",
    "CAP_SETUID", "CAP_SETPCAP", "CAP_LINUX_IMMUTABLE",
    "CAP_NET_BIND_SERVICE", "CAP_NET_BROADCAST", "CAP_NET_ADMIN",
    "CAP_NET_RAW", "CAP_IPC_LOCK", "CAP_IPC_OWNER",
    "CAP_SYS_MODULE", "CAP_SYS_RAWIO", "CAP_SYS_CHROOT",
    "CAP_SYS_PTRACE", "CAP_SYS_PACCT", "CAP_SYS_ADMIN",
    "CAP_SYS_BOOT", "CAP_SYS_NICE", "CAP_SYS_RESOURCE",
    "CAP_SYS_TIME", "CAP_SYS_TTY_CONFIG", "CAP_MKNOD",
    "CAP_LEASE", "CAP_AUDIT_WRITE", "CAP_AUDIT_CONTROL",
    "CAP_SETFCAP", "CAP_MAC_OVERRIDE", "CAP_MAC_ADMIN",
    "CAP_SYSLOG", "CAP_WAKE_ALARM", "CAP_BLOCK_SUSPEND",
    "CAP_AUDIT_READ", "CAP_PERFMON", "CAP_BPF", "CAP_CHECKPOINT_RESTORE"
};

void map(uint64_t *bitmask)
{
   printf("Capabilities:\n");
   for (int i = 0; i < 41; i++) {
       if ((*bitmask) & (1ULL << i)) {
           printf("%s\n", cap_names[i]);
       }
   }
}
```

As you can see, I first spell out a `char` pointer to an array of capabilities, ensuring their order matches with the Linux kernel source code at ![`linux/include/linux/capability.h`](https://github.com/torvalds/linux/blob/master/include/linux/capability.h).

Inside the `map()` function (called by `main()`), I take in a 64-bit bitmask, and for the 41 capabilities listed, if a **bitwise `AND`** between the passed bitmask and a left-shifted `1` shifted over by the amount of iterations passed match - then that specific bit in the bitmask is enabled, and I should print the capability associated with the iteration.

To better illustrate this point, let's use a simple bitmask of `0x2`, or `10` in binary, or in a 64-bit representation:
```
0000000000000000000000000000000000000000000000000000000000000010
```

In the first loop, we do a bitwise `AND` of 
```
...0010
```
and
```
...0001
```
which results in 
```
...0000
```

and no capability is printed.

In the next loop, we do a bitwise `AND` of 
```
...0010
```
and
```
...0010
```
which results in
```
...0010
```

Since enabled bit matches, we print the capability associated with this iteration: `cap_names[1]`, or `CAP_DAC_OVERRIDE`.


View more on GitHub: ![bitcap](https://github.com/sinclairsecurity/bitcap/tree/master)