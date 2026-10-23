# LAB 1 – Report

Zhuchkova Sofia FAF-241

## Part 1

1. Who owns the files you just created, and which group?

The files are owned by `vboxuser` and belong to the `vboxuser` group.

Terminal output:
```
vboxuser@Name:~/os-lab1$ ls -l
total 12
drwxrwxr-x 2 vboxuser vboxuser 4096 Oct 19 13:43 demo
-rw-rw-r-- 1 vboxuser vboxuser   24 Oct 22 20:50 note.txt
-rw-rw-r-- 1 vboxuser vboxuser   24 Oct 22 20:50 renamed.txt
```

2. What do the ten characters at the start of `ls -l` mean for `note.txt` after `chmod 600`?

The characters are `-rw-------`. The first `-` means it is a regular file. The next three characters, `rw-`, mean the owner can read and edit the file. The remaining six `-` characters mean that the group and other users have no permissions.

Terminal output:
```
vboxuser@Name:~/os-lab1$ chmod 600 note.txt
vboxuser@Name:~/os-lab1$ ls -l note.txt
-rw------- 1 vboxuser vboxuser 24 Oct 22 20:50 note.txt
```

3. Name two directories directly under `/` and say, in one line each, what they hold.

- `/home` — contains users' personal directories and files.
- `/etc` — contains system configuration files.

Terminal output:
```
vboxuser@Name:~/os-lab1$ ls -la /
drwxr-xr-x  20 root root  4096 Sep 28 15:34 .
drwxr-xr-x 141 root root 12288 Oct 19 13:00 etc
drwxr-xr-x   3 root root  4096 Sep 28 16:08 home
```

## Part 2

1. What is the PID of process number 1, and what is it?

The PID is `1`, and the process is `systemd`. It starts and manages system services when Linux boots.

Terminal output:
```
vboxuser@Name:~/os-lab1$ ps aux | head
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.2  26628  9132 ?        Ss   20:34   0:04 /usr/lib/systemd/systemd --switched-root --system --deserialize=51
```

2. Roughly how many processes were running on your idle VM?

Approximately 250 processes were running on my VM. The exact number can change as processes start and stop.

Terminal output:
```
vboxuser@Name:~/os-lab1$ ps aux | wc -l
255
```

The output includes the header line, so this represents approximately 254 processes.

3. From `/proc/<pid>/status`, what does the "State:" line say for your sleeping process?

The state is `S (sleeping)`, which means the process is waiting and isn't currently doing work.

Terminal output:
```
vboxuser@Name:~/os-lab1$ sleep 300 &
[1] 9341
vboxuser@Name:~/os-lab1$ grep '^State:' /proc/9341/status
State:  S (sleeping)
```

## Part 3

1. How much total RAM does the VM have, and how much is free?

The VM has 4.1 GiB of RAM in total, with around 180 MiB free at the time of checking.

Terminal output:
```
vboxuser@Name:~/os-lab1$ free -h
               total        used        free      shared  buff/cache   available
Mem:           4.1Gi       3.7Gi       180Mi        91Mi       443Mi       443Mi
Swap:             0B          0B          0B
```

2. What is swap, and how much is configured?

Swap is space on a disk that Linux can use as extra memory when RAM is running low. My VM has 0 B of swap configured.

Terminal output:
```
vboxuser@Name:~/os-lab1$ free -h
Swap:             0B          0B          0B
```

3. How much resident memory (VmRSS) does a bare `sleep` process use, and does that surprise you?

The `sleep` process uses 7892 kB of resident memory (about 7.7 MiB). This is a small amount, which makes sense because the program only waits for a specified time and does very little work.

Terminal output:
```
vboxuser@Name:~/os-lab1$ sleep 300 & PID=$!
[1] 9553
vboxuser@Name:~/os-lab1$ grep VmRSS /proc/$PID/status
VmRSS:        7892 kB
vboxuser@Name:~/os-lab1$ kill $PID
[1]+  Terminated                 sleep 300
```

## Part 4

1. Which device is your root filesystem `/` mounted on?

The root filesystem is mounted on `/dev/sda2`, which is a partition of the VM's virtual disk.

Terminal output:
```
vboxuser@Name:~/os-lab1$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        49G  6.9G   40G  15% /
```

2. Give one entry from `/dev` and say what real thing it stands for.

`/dev/sda` represents the VM's virtual hard disk.

Terminal output:
```
vboxuser@Name:~/os-lab1$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0    50G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    50G  0 part /
```

3. In one sentence: what does "everything is a file" mean, based on what you saw?

It means Linux lets you access many things, including files, disks, and devices, through a common file-based system.

## Wrap up
The operating system manages four main resources:

1. CPU and processes — `ps aux` displays the running processes.
2. Memory — `free -h` shows RAM and swap usage.
3. Storage and filesystems — `df -h` shows disk space and mounted filesystems.
4. Devices — `lsblk` lists block devices, such as disks and partitions.
