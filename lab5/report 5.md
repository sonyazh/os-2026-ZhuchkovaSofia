# LAB 5 – Report

Zhuchkova Sofia FAF-241

## Purpose of the work

To explore the Linux file system by hand: the directory tree and the file types, inodes and the difference between hard and symbolic links, permissions, and the disks and kernel information that Linux shows as files.

## Part 1. The tree and the file types

```bash
whoami
mkdir -p ~/os-lab5 && cd ~/os-lab5
ls /
ls -ld /etc /etc/passwd /bin /dev/null /tmp
ls -l /dev | head
file /usr/bin/ls /etc/passwd /bin
du -sh /var/log 2>/dev/null ; du -sh /usr 2>/dev/null
```

Output of the second line:

```
lrwxrwxrwx   1 root root     7 Apr 20  2026 /bin -> usr/bin
crw-rw-rw-   1 root root  1, 3 Nov 23 09:47 /dev/null
drwxr-xr-x 141 root root 12288 Oct 24 14:13 /etc
-rw-r--r--   1 root root  2778 Sep 28 16:08 /etc/passwd
drwxrwxrwt  16 root root   360 Nov 23 09:55 /tmp
```

Output of `file` and `du`:

```
/usr/bin/ls: symbolic link to ../lib/cargo/bin/coreutils/ls
/etc/passwd: ASCII text
/bin:        symbolic link to usr/bin

316M	/var/log
5.3G	/usr
```

### Observe

**1. For each path in the second line, write its type from the first letter of `ls -l`.**

| Path | First letter | Type |
|---|---|---|
| `/etc` | `d` | directory |
| `/etc/passwd` | `-` | regular file |
| `/bin` | `l` | symbolic link |
| `/dev/null` | `c` | character device |
| `/tmp` | `d` | directory|

**2. What is `/bin` on your system: a directory or a link, and to where?**

It is a symbolic link (`l`), and it points to `usr/bin` (`/bin -> usr/bin`). `file /bin` says the same: "symbolic link to usr/bin". So on this system the old `/bin` directory is just another name for `/usr/bin`.

**3. Find one character device and one block device in /dev.**

- Character device: `/dev/null` (first letter `c`, major and minor numbers `1, 3`). `/dev/console` and `/dev/autofs` are also character devices.
- Block device: `/dev/loop0` (first letter `b`, major and minor numbers `7, 0`):

```
brw-rw---- 1 root disk 7, 0 Nov 23 11:27 loop0
```

Character devices are read and written as a stream of bytes, block devices (disks) are read and written in blocks.

A loop device is a virtual disk backed by a file. There are many on this system (`loop0`, `loop1`, `loop10`...).

**4. Which is bigger on your laptop, `/var/log` or `/usr`, and by roughly how much?**

`/usr` is bigger: 5.3 GB against 316 MB for `/var/log`, so `/usr` is about 17 times bigger. That makes sense, because `/usr` holds almost all the installed programs and libraries, while `/var/log` only holds log files.

## Part 2. Inodes and links

### Steps 1 and 2: inode, then a hard link

```
$ echo hello > notes.txt
$ stat notes.txt
  File: notes.txt
  size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: 8,2	Inode: 1339093     Links: 1
...
$ ls -li notes.txt
1339093 -rw-rw-r-- 1 vboxuser vboxuser 6 Nov 23 11:41 notes.txt
$ ln notes.txt hard.txt
$ ls -li notes.txt hard.txt
1339093 -rw-rw-r-- 2 vboxuser vboxuser 6 Nov 23 11:41 hard.txt
1339093 -rw-rw-r-- 2 vboxuser vboxuser 6 Nov 23 11:41 notes.txt
$ rm notes.txt
$ cat hard.txt ; ls -li hard.txt
hello
1339093 -rw-rw-r-- 1 vboxuser vboxuser 6 Nov 23 11:41 hard.txt
```

**Observe 1. Which number in `stat` is the inode, and which tells how many names the file has?**

`Inode: 1339093` is the inode number, the file's real identity on the disk. `Links: 1` is the number of names (hard links) the file has. After `ln notes.txt hard.txt` it became 2.

**Observe 2. After `rm notes.txt`, why could you still read `hard.txt`? What does `rm` really remove?**

`notes.txt` and `hard.txt` were two names for the same inode (both showed 1339093), so the content was never copied. `rm` doesn't delete the file itself, it only removes one name from the directory and lowers the link count (2 to 1). The data is freed only when the link count reaches 0. Since `hard.txt` still pointed to inode 1339093, `cat hard.txt` still printed `hello`.

### Step 3: symbolic link, then remove its target

```
$ ln -s hard.txt soft.txt
$ ls -li hard.txt soft.txt
1339093 -rw-rw-r-- 1 vboxuser vboxuser 6 Nov 23 11:41 hard.txt
1339175 lrwxrwxrwx 1 vboxuser vboxuser 8 Nov 23 12:03 soft.txt -> hard.txt
$ cat soft.txt
hello
$ rm hard.txt
$ cat soft.txt ; ls -l soft.txt
cat: soft.txt: No such file or directory
lrwxrwxrwx 1 vboxuser vboxuser 8 Nov 23 12:03 soft.txt -> hard.txt
```

### Step 4: directories have links too

My prediction for the link count: probably at least one? Since it is a directory it must link at least to itself.

```
$ mkdir d ; ls -ld d
drwxrwxr-x 2 vboxuser vboxuser 4096 Nov 23 12:11 d
$ mkdir d/x d/y ; ls -ld d
drwxrwxr-x 4 vboxuser vboxuser 4096 Nov 23 12:11 d
```

### Step 5: size is not space

```
$ echo x > one.txt ; ls -l one.txt ; du -h one.txt
-rw-rw-r-- 1 vboxuser vboxuser 2 Nov 23 12:11 one.txt
4.0K	one.txt
$ truncate -s 1G big.img ; ls -lh big.img ; du -h big.img
-rw-rw-r-- 1 vboxuser vboxuser 1.0G Nov 23 12:11 big.img
0	big.img
$ rm big.img
```

### Observe (continued)

**3. Why did `cat soft.txt` fail at the end, although `soft.txt` still exists? How big is `soft.txt` and why exactly that size?**

A symbolic link is a separate small file with its own inode (1339175, not 1339093) that stores a path, here the text `hard.txt`. When `cat` opens `soft.txt`, the system follows that path. After `rm hard.txt` there is no file with that name any more (the link count of its inode dropped to 0 and the data was freed), so the path leads nowhere and `cat` reports "No such file or directory". The link itself still exists, it is just broken (dangling).

`soft.txt` is 8 bytes because the link stores the text of the path, and `hard.txt` has 8 characters.

**4. A new directory has 2 links, and 4 after you create two subdirectories. Where does each link come from?**

An empty directory `d` has 2 links: one is its name `d` in the parent directory, and the other is the `.` entry inside `d` itself (which points to `d`). Each subdirectory has a `..` entry that points back to its parent, so `d/x` and `d/y` add one link each to `d`: 2 + 2 = 4.

**5. Why does a 2-byte file use 4.0K, and a 1 GB file use 0?**

The file system gives disk space in whole blocks of 4 KB, so even the 2-byte file (`x` and a newline) takes one full block, and `du` shows the space actually used on disk (4.0K). `truncate -s 1G` only sets the file size to 1 GB without writing any data, which makes a sparse file: no blocks are allocated, so `du` shows 0. `ls -l` shows the size of the file as it looks to programs, `du` shows what it really takes on disk.

## Part 3. Permissions

### Step 6: files

```
$ echo secret > s.txt
$ chmod 600 s.txt ; ls -l s.txt
-rw------- 1 vboxuser vboxuser 7 Nov 23 12:35 s.txt
$ chmod 000 s.txt ; cat s.txt
cat: s.txt: Permission denied
$ chmod u+r s.txt ; cat s.txt
secret
$ umask ; touch new.txt ; mkdir newdir ; ls -ld new.txt newdir
0002
-rw-rw-r-- 1 vboxuser vboxuser    0 Nov 23 12:36 new.txt
drwxrwxr-x 2 vboxuser vboxuser 4096 Nov 23 12:36 newdir
```

### Step 7: directories

My prediction: 
1. we create a folder and a file with the 'hi' text inside.
2. we try to access it, so ls should print the properties and the cat should print 'hi'?
3. the only difference is `u=x`... I have no idea.
4. we probably assign a new value to the box? Maybe a new permission?

```
$ mkdir box ; echo hi > box/f
$ chmod u=r,go= box ; ls box ; cat box/f
f
cat: box/f: Permission denied
$ chmod u=x,go= box ; ls box ; cat box/f
ls: cannot open directory 'box': Permission denied
hi
$ chmod 755 box
```

### Observe

**1. Write 600, 644, 755 and 700 as rwx strings, and give one real use for each.**

| Number | String | Real use |
|---|---|---|
| 600 | `rw-------` | private file only the owner can read and write, like an SSH private key |
| 644 | `rw-r--r--` | normal file that everybody may read, like `/etc/passwd` |
| 755 | `rwxr-xr-x` | programs and directories everybody may run or enter, like `/usr/bin/ls` |
| 700 | `rwx------` | private directory, like `~/.ssh` |

Each digit is the sum of r = 4, w = 2, x = 1, for the owner, the group and the others in that order. For example 6 = 4 + 2 = `rw-`, 5 = 4 + 1 = `r-x`.

**2. With `chmod 000` you own the file but cannot read it. Why does ownership not help?**

Being the owner doesn't give access by itself. The system checks the permission bits of the class that fits you (owner, group or others). With 000 the owner's bits are all off, so `cat` was refused: "Permission denied". What ownership does give is the right to change the permissions: I ran `chmod u+r s.txt` and then could read the file again.

**3. What is your umask, and how does it explain the permissions of `new.txt` and `newdir`?**

My umask is `0002`. It lists the permissions that are **removed** from new files and directories. New files start at 666 (`rw-rw-rw-`, files don't get `x` by default) and new directories at 777. Removing 002 (the write bit for others) gives 664 (`rw-rw-r--`) for `new.txt` and 775 (`rwxrwxr-x`) for `newdir`, which matches the output.

**4. Explain the `box` experiment: what does `r` allow on a directory, and what does `x` allow?**

`r` on a directory allows reading the list of names inside it. `x` allows going through the directory to reach the files inside it, when you know their names.

With `r` only, `ls box` worked and showed `f`, but `cat box/f` failed because I couldn't go through the directory. With `x` only, `ls box` failed because I couldn't read the list of names, but `cat box/f` worked, because I could go through the directory and the file `f` itself was still readable.

### Step 8: special bits and other people's files

```
$ ls -l /usr/bin/passwd ; ls -ld /tmp
-rwsr-xr-x 1 root root 93640 Feb  2  2026 /usr/bin/passwd
drwxrwxrwt 15 root root 340 Nov 23 13:11 /tmp
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1314 Sep 28 16:03 /etc/shadow
```

**5. What do the `s` in `/usr/bin/passwd` and the `t` in `/tmp` mean, and why does `passwd` need the `s`?**

The `s` is the setuid bit. Normally a program runs with the permissions of the user who starts it. A setuid program runs with the permissions of the **owner of the program file**, which here is root.

`passwd` needs it because it must change my password, which is stored in `/etc/shadow`. That file is owned by root and only root can write to it (`-rw-r-----`), so I can't even read it (the `cat` was refused). Because `passwd` runs as root, it can update my line in the file, and nothing else in it, even though I am an ordinary user.

The `t` on `/tmp` is the sticky bit. `/tmp` is writable by everyone (`rwxrwxrwx`), so everybody can create files there. With the sticky bit, a user can delete or rename only the files they own, so one user can't remove another user's temporary files.

## Part 4. Disks and the kernel as files

```
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0     4K  1 loop /snap/bare/5
...(loop1 to loop19, all mounted under /snap/...)
sda      8:0    0    50G  0 disk 
├─sda1   8:1    0     1M  0 part 
└─sda2   8:2    0    50G  0 part /
sr0     11:0    1  1024M  1 rom  
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           846M  2.4M  844M   1% /run
/dev/sda2        49G  9.9G   37G  22% /
tmpfs           2.1G     0  2.1G   0% /dev/shm
none            1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs           2.1G  8.0K  2.1G   1% /tmp
none            1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
tmpfs           423M   84K  423M   1% /run/user/1000
$ mount | grep " / "
/dev/sda2 on / type ext4 (rw,relatime)
$ grep -c processor /proc/cpuinfo
4
$ head -3 /proc/meminfo
MemTotal:        4330576 kB
MemFree:          173204 kB
MemAvailable:    1357684 kB
$ head -5 /proc/self/status
Name:	head
Umask:	0002
State:	R (running)
Tgid:	5693
Ngid:	0
$ ls -l /proc/self/fd
total 0
lrwx------ 1 vboxuser vboxuser 64 Nov 23 13:18 0 -> /dev/pts/0
lrwx------ 1 vboxuser vboxuser 64 Nov 23 13:18 1 -> /dev/pts/0
lrwx------ 1 vboxuser vboxuser 64 Nov 23 13:18 2 -> /dev/pts/0
lr-x------ 1 vboxuser vboxuser 64 Nov 23 13:18 3 -> /proc/5705/fd
```

### Observe

**1. Draw your disk: disk name, partitions, which one is mounted at `/` and with what file system type.**

My system is a virtual machine, so the "laptop" disk is a 50 GB virtual disk:

```
sda (disk, 50 GB)
├── sda1 (partition, 1 MB, not mounted)
└── sda2 (partition, 50 GB, mounted at /, file system type ext4)
```

`mount | grep " / "` shows `/dev/sda2 on / type ext4`. The many `loop` devices are small virtual disks, each backing one snap package (`/snap/...`), and `sr0` is the empty CD/DVD drive.

**2. How full is your root file system?**

`/dev/sda2` has 49 GB, with 9.9 GB used (22%) and 37 GB free.

**3. How many CPU cores and how much RAM does your laptop have, according to `/proc`?**

4 CPU cores (`grep -c processor /proc/cpuinfo` printed 4) and about 4.1 GB of RAM (`MemTotal: 4330576 kB`, which is about 4.1 GiB). These are the values given to the virtual machine, not the physical ones of the host computer.

**4. `/proc/self/status` showed the name of a program. Which one, and why that one?**

It showed `Name: head`. `/proc/self` always means "the process that is reading it", and here the process reading the file was `head`. If I had used `cat`, it would have shown `cat`.

The files in `/proc` are not stored on the disk. The kernel creates their content at the moment a program reads them.

**Extra note:** in `/proc/self/fd`, the file descriptors 0, 1 and 2 (standard input, output and error) all point to the terminal `/dev/pts/0`. Descriptor 3 is the directory `ls` itself opened to read the list.

## Stretch

### (a) Magic numbers

```
$ gzip -c /etc/passwd > notes.txt ; file notes.txt ; xxd notes.txt | head -1
notes.txt: gzip compressed data, was "passwd", last modified: Mon Sep 28 16:08:40 2026, from Unix, original size modulo 2^32 2778
00000000: 1f8b 0808 0891 ba6a 0003 7061 7373 7764  .......j..passwd
```

**Which bytes tell `file` the real type?**

The first two bytes, `1f 8b`, are the magic number of the gzip format. `file` reads the beginning of the file and recognises this, so it ignores the name `notes.txt` and reports "gzip compressed data". (The original name `passwd` is also stored inside the gzip header, which is why it is visible at the end of the line.) The original size, 2778 bytes, matches `/etc/passwd`.

### (b) Hunt for setuid programs

```
$ find / -xdev -perm -4000 -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/cargo/bin/sudo
/usr/lib/cargo/bin/su
/usr/sbin/pppd
/usr/bin/passwd
/usr/bin/su
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/sudo.ws
/usr/bin/fusermount3
/usr/bin/chfn
/usr/bin/mount
/usr/bin/pkexec
/usr/bin/ntfs-3g
/usr/bin/umount
```

The command searches the root file system for regular files with the setuid bit (`-perm -4000`). Besides the well-known ones (`passwd`, `su`, `sudo`, `mount`, `umount`), my laptop also has these, which I did not see on the lecture slide: `pkexec` (runs a command as another user), `fusermount3` (lets normal users mount FUSE file systems), `ntfs-3g` (mounts Windows NTFS disks), `pppd` (network connections), `ssh-keysign`, `chsh` and `chfn` (change my shell and my user info), `gpasswd` (group administration) and `dbus-daemon-launch-helper`. Each of them needs root privileges for one specific job, which is why they have the setuid bit. The list should stay short, because every setuid program is a possible way to gain root rights if it has a bug.
