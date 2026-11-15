# Operating Systems: A Technician's Guide

The OS is not a black box. It is a kernel that shares the CPU, memory and disks between programs, plus the tools to see what it is doing. This guide uses Linux and shows the commands you use when a machine is slow, full, or won't start.

---

## 1. The 30-Second Mental Model

Think of the OS as a building manager. Programs (tenants) never touch the hardware directly. They ask the **kernel** for CPU time, memory and disk access, and the kernel decides who gets what and who is allowed in.

```text
  Applications   (nginx, python, your shell)
        |  system calls
  +-----v------------------------------+
  |  Kernel                            |
  |  scheduler | memory mgr | file sys | drivers
  +-----+------------------------------+
        |
  Hardware: CPU, RAM, disk, NIC
```

Everything the system does is a **process** with a PID. Everything on disk is a **file** with an owner and permissions. Learn to inspect those two things and you can diagnose most problems.

---

## 2. How It Works in Practice

### 2.1 The Boot Process

When you press the power button the machine passes through a chain, and each stage starts the next:

```text
Firmware (BIOS/UEFI)  -> POST, finds a boot device
Bootloader (GRUB)     -> shows the menu, loads the kernel
Kernel + initramfs    -> detects hardware, mounts the root filesystem
systemd (PID 1)       -> starts services until the default target is reached
Login prompt / desktop
```

- **BIOS** is the older firmware and reads the boot code from the first sector of the disk (MBR). **UEFI** is the modern replacement and loads a bootloader file from the EFI System Partition.
- The **initramfs** is a small temporary filesystem that lets the kernel load the drivers it needs to reach the real disk.
- **systemd** is the first process (PID 1). It starts everything else.

```bash
systemd-analyze                  # total boot time
systemd-analyze blame            # slowest units at boot
systemctl get-default            # default target (e.g. graphical.target)
journalctl -b                    # logs from the current boot
journalctl -b -1                 # logs from the previous boot
```

### 2.2 Processes and CPU

A program on disk becomes a process when it runs. The **scheduler** gives each runnable process short slices of CPU time.

```bash
ps aux                           # every process, with user, CPU %, memory %
ps aux --sort=-%cpu | head       # top CPU users
top                              # live view (q to quit)
htop                             # friendlier live view, if installed
uptime                           # load average over 1, 5, 15 minutes
nproc                            # number of CPU cores
```

**Load average** is the average number of processes running or waiting. As a rule of thumb, a load persistently above your core count means the CPU is saturated.

Stopping a process:
```bash
kill <PID>                       # polite request (SIGTERM)
kill -9 <PID>                    # force kill (SIGKILL), last resort
systemctl status nginx           # manage services instead of killing them
```

### 2.3 Memory

RAM is fast and limited. When it runs short, the kernel moves rarely used pages to **swap** on disk, which is far slower. The kernel also uses spare RAM as a file cache, so "used" memory is not always a problem.

```bash
free -h                          # look at the "available" column, not "free"
vmstat 1 5                       # si/so columns show swapping activity
ps aux --sort=-%mem | head       # top memory users
```

### 2.4 Disks, File Systems and Permissions

Disks are split into partitions, each formatted with a **file system** (ext4, xfs, NTFS) and **mounted** at a directory. Every file has an **inode** that stores its metadata (owner, permissions, size, disk location). The filename is just a pointer to that inode.

```bash
lsblk                            # disks and partitions as a tree
df -h                            # space used per mounted file system
df -i                            # inode usage per file system
du -sh /var/* | sort -h          # which directory is using the space
mount | grep ' / '               # how root is mounted
```

**Permissions** have three groups (owner, group, others) and three rights (read, write, execute).

```text
-rwxr-x---  1 lukas devs  4096 Nov 15 report.sh
 |||||||||
 |||||||++- others: none
 |||||+---- group:  r-x  (read, execute)
 ||+------- owner:  rwx  (read, write, execute)
 +--------- file type (- file, d directory)
```

Numeric form: r=4, w=2, x=1. So `rwxr-x---` is `750`.

```bash
ls -l                            # show permissions and owner
chmod 750 report.sh              # set permissions
chmod u+x report.sh              # add execute for the owner
chown lukas:devs report.sh       # change owner and group
id                               # who am I, which groups
```

---

## 3. What Happens When It Breaks

| Symptom | Likely cause | First commands |
|---------|--------------|----------------|
| Machine won't boot | Bad GRUB entry, missing kernel, full or corrupt root disk | Bootloader menu, recovery mode, then `journalctl -b -1` |
| Boots into emergency mode | Bad `/etc/fstab` entry or failed disk mount | `journalctl -xb`, check `/etc/fstab` |
| Slow boot | A service waiting on a timeout | `systemd-analyze blame` |
| System is slow, CPU at 100% | Runaway process | `top`, `ps aux --sort=-%cpu` |
| Slow and the disk light is constantly on | Swapping, out of RAM | `free -h`, `vmstat 1` |
| Process disappears suddenly | OOM killer ended it | `journalctl -k \| grep -i "out of memory"` |
| `No space left on device` | Disk full | `df -h`, then `du -sh /var/* \| sort -h` |
| `No space left` but `df -h` shows free space | Out of inodes (many tiny files) | `df -i` |
| `Permission denied` | Wrong owner, mode, or user | `ls -l`, `id`, then `chmod`/`chown` |
| `Command not found` after install | Not on `PATH` | `echo $PATH`, `which <cmd>` |
| Service won't start | Config error, port in use | `systemctl status <svc>`, `journalctl -u <svc>` |

**Diagnostic order for a slow machine:** check CPU (`uptime`, `top`), then memory (`free -h`, `vmstat`), then disk (`df -h`, `iostat` if installed). Find which resource is exhausted before changing anything.

---

## 4. Hands-On Lab

Use a Linux VM, WSL, or a free-tier cloud instance. These steps only read system state, except where noted.

1. **Read the boot.**
   ```bash
   systemd-analyze
   systemd-analyze blame | head
   ```
   Which service took longest?
2. **Find PID 1.**
   ```bash
   ps -p 1 -o pid,comm
   ```
   You should see `systemd`.
3. **Create a CPU load and find it.** In one terminal:
   ```bash
   yes > /dev/null &
   ```
   In the same terminal, run `top`. Find `yes` using nearly 100% of one core, then stop it with `kill %1`.
4. **Inspect memory.**
   ```bash
   free -h
   ps aux --sort=-%mem | head -5
   ```
   Note the difference between `free` and `available`.
5. **Explore disks.**
   ```bash
   lsblk
   df -h
   df -i
   ```
6. **Practice permissions** (safe, in a temp directory):
   ```bash
   mkdir ~/lab && cd ~/lab
   echo 'echo hello' > hello.sh
   ls -l hello.sh          # no execute bit
   ./hello.sh              # Permission denied
   chmod u+x hello.sh
   ./hello.sh              # prints hello
   chmod 000 hello.sh
   cat hello.sh            # Permission denied again
   cd ~ && rm -r ~/lab     # clean up
   ```
7. **Check yourself:** when `./hello.sh` failed, which of the three permission groups and which right were missing?

---

## Further Reading
- [Linux Boot Process: Step-by-Step Guide](https://www.magnetbyte.com/linux-boot-process-step-by-step)
- [UEFI, GRUB, initramfs & systemd](https://www.golinuxcloud.com/linux-boot-process-explained-step-detail/)
- [Understand GRUB, initramfs, and systemd failures](https://commandinline.com/linux-boot-process-grub-systemd/)
- [Unix and Internet Fundamentals HOWTO](https://www.linuxdoc.org/HOWTO/Unix-and-Internet-Fundamentals-HOWTO.html)
