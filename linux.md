# Linux & DevOps Command Notes

These are my Linux / DevOps lab notes, cleaned up and organised. Every command has a short definition. The lab tasks are in their own section, with steps.

> Lab machine used in these notes: `ubuntu@devops-server` (Ubuntu, 2 CPUs, kernel 7.0.0-43-generic)

---

## Table of Contents

1. [Storage: Mounting & LVM](#1-storage-mounting--lvm)
2. [Navigating the Filesystem](#2-navigating-the-filesystem)
3. [Finding Files and Commands](#3-finding-files-and-commands)
4. [Shell Productivity](#4-shell-productivity)
5. [Redirection, Pipes, tee & Exit Codes](#5-redirection-pipes-tee--exit-codes)
6. [System Monitoring & Performance](#6-system-monitoring--performance)
7. [Troubleshooting Labs (Tasks)](#7-troubleshooting-labs-tasks)
8. [Practice Task Checklist](#8-practice-task-checklist)
9. [Command Index (A–Z)](#9-command-index-az)

---

## 1. Storage: Mounting & LVM

### 1.1 Temporary vs persistent mounting

| Type | How | Lasts |
|---|---|---|
| **Temporary mounting** | `mount <device> <directory>` | Until you run `umount` or the machine reboots |
| **Persistent mounting** | Add a line to `/etc/fstab`, then test with `mount -a` | Survives reboots; mounted automatically at boot |

- **`mount`**: Attaches a filesystem (a disk, a partition, an LV or a network share) to a directory, so its files appear in that directory.
- **`umount`**: Detaches a mounted filesystem.
- **`/etc/fstab`**: The "filesystem table". Each line lists something to mount at boot: `device  mount-point  fs-type  options  dump  pass`.
- **`mount -a`**: Mounts everything listed in `/etc/fstab` that is not already mounted. Run it after editing fstab: if it prints an error, fix the line **before** rebooting, because a broken fstab can stop the machine from booting.

### 1.2 LVM concepts (Logical Volume Manager)

LVM sits between your disks and your filesystems. You can pool disks together and resize volumes without repartitioning.

| Layer | Meaning | Analogy |
|---|---|---|
| **PV** — Physical Volume | A disk or partition prepared for LVM; raw space | The ingredients |
| **VG** — Volume Group | One or more PVs combined into a single pool of space | The mixing bowl |
| **LV** — Logical Volume | A slice of the VG that you format and mount like a normal partition | The serving you take out of the bowl |

```
/dev/sdb1  /dev/sdb2  /dev/sdb3     <- PVs  (pvcreate)
      \        |        /
          vg1 (pool)                <- VG   (vgcreate / vgextend)
        /           \
   lv1 (400M)    free space         <- LVs  (lvcreate / lvextend)
      |
   mkfs.xfs -> mount -> /data
```

### 1.3 TASK — Build an LVM volume from scratch

**Goal:** Turn a blank disk (`/dev/sdb`) into a mounted, permanent LVM volume.

**Step 0: Install the LVM tools (if they are missing)**
```bash
sudo apt install lvm2
```

**Step 1: Partition the disk with `parted`**
```bash
sudo parted /dev/sdb
(parted) mklabel gpt                     # create a new GPT partition table (this erases existing partitions)
(parted) mkpart primary 1MiB 5GiB        # create partition 1, from 1 MiB to 5 GiB
(parted) set 1 lvm on                    # flag partition 1 as an LVM partition
(parted) print                           # show the partition table to check it
(parted) quit                            # leave parted
```

**Step 2: Turn the partition into a Physical Volume**
```bash
sudo pvcreate /dev/sdb1
```

**Step 3: Create the Volume Group (the pool)**
```bash
sudo vgcreate vg1 /dev/sdb1
```

**Step 4: Carve out a Logical Volume**
```bash
sudo lvcreate -n lv1 -L 600M vg1         # -n = name, -L = size
```

**Step 5: Format it with a filesystem**
```bash
sudo mkfs.xfs /dev/vg1/lv1
```

**Step 6: Create a mount point and mount it**
```bash
sudo mkdir /data
sudo mount /dev/vg1/lv1 /data
```

**Step 7: Make the mount permanent.** Add this line to `/etc/fstab`:
```
/dev/vg1/lv1   /data   xfs   defaults   0 0
```

**Step 8: Test fstab and verify**
```bash
sudo mount -a          # no output = fstab is OK
lsblk                  # the LV should show under sdb1 with mountpoint /data
df -h /data            # shows the size and usage of the new filesystem
```

### 1.4 LVM and disk commands — definitions

| Command | Definition |
|---|---|
| `lsblk` | Lists block devices (disks, partitions, LVs) as a tree, with sizes and mount points |
| `free -h` | Shows RAM and swap usage. `-h` = human-readable (MiB/GiB) |
| `df -h` | Shows disk space used and free on each **mounted filesystem** |
| `df -h <path>` | Shows the space for the one filesystem that contains `<path>` |
| `apt install lvm2` | Installs the LVM tools package (`pvcreate`, `vgcreate`, `lvcreate`, …) |
| `parted /dev/sdb` | Interactive tool to create and edit partition tables and partitions |
| `pvcreate <partition…>` | Initialises one or more partitions or disks as LVM Physical Volumes |
| `pvdisplay` | Shows details of all PVs: size, which VG they belong to, free space |
| `pvmove <PV>` | Moves all data off one PV onto the other PVs in the same VG. Use it before removing a disk; the other PVs need enough free space |
| `vgcreate <vg> <PV…>` | Creates a new Volume Group from one or more PVs |
| `vgextend <vg> <PV>` | Adds another PV to an existing VG, making the pool bigger |
| `vgreduce <vg> <PV>` | Removes an (empty) PV from a VG. Usually run after `pvmove` |
| `vgdisplay` | Shows details of VGs: total size, free space, number of PVs/LVs |
| `lvcreate -n <name> -L <size> <vg>` | Creates a Logical Volume with a given name and size inside a VG |
| `lvdisplay` | Shows details of all LVs: path, size, status |
| `lvextend <LV> -L <size>` | Makes an LV bigger (see the note below) |
| `mkfs.xfs <device>` | Formats a device with the XFS filesystem (this erases its data) |
| `mkdir <dir>` | Creates a directory (here, used as a mount point) |
| `mount <device> <dir>` | Mounts the device on the directory |

### 1.5 Resizing an LV — important notes

- **`-L 800M` vs `-L +800M`**
  - `lvextend /dev/vg1/lv1 -L 800M` sets the LV to **exactly 800M** in total.
  - `lvextend /dev/vg1/lv1 -L +800M` **adds** 800M to the current size.
- **Grow the filesystem too.** `lvextend` only grows the *volume*. The filesystem inside it stays the same size until you grow it, so `df -h` will still show the old size. Either:
  ```bash
  sudo lvextend -r -L +800M /dev/vg1/lv1   # -r = resize the filesystem at the same time
  # or, after a plain lvextend on XFS:
  sudo xfs_growfs /data                    # grow XFS, using its mount point
  ```
- **XFS can grow but cannot shrink.** If you need to shrink volumes, use ext4 (`mkfs.ext4`, `resize2fs`).

### 1.6 Mistakes spotted in my command history (and the fixes)

| What I typed | Problem | Correct |
|---|---|---|
| `pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb2` (before installing lvm2) | `lvm2` was not installed yet, and `sdb2` was listed twice | `apt install lvm2`, then `pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb3` |
| `dh -h` | Typo; no such command | `df -h` |
| `df-h /mountlv` | Missing space | `df -h /mountlv` |
| `df -h /mountlv` vs `df -h mountlv` | `/mountlv` means a folder at the root of the system. `mountlv` (no slash) means the folder in my **current** directory, which is where I created it with `mkdir mountlv` | Use `df -h ~/mountlv` or `df -h mountlv` |

### 1.7 Mounting an NFS share

**NFS** (Network File System) lets you mount a folder that lives on another machine.

My notes had `Mount -t nfs 192.168.1.235`. That is incomplete: the export path and the local mount point are missing, and Linux commands are lowercase. The correct form is:

```bash
sudo apt install nfs-common                          # NFS client tools
showmount -e 192.168.1.235                           # list the folders the server exports
mkdir mountnfs                                       # local mount point
sudo mount -t nfs 192.168.1.235:/path/to/export mountnfs
```

- **`mount -t nfs`**: `-t` sets the filesystem type, here `nfs`.
- **`showmount -e <server>`**: Lists the directories an NFS server shares.
- **Persistent NFS mount**: an `/etc/fstab` line such as
  `192.168.1.235:/path/to/export  /mnt/nfs  nfs  defaults,_netdev  0 0`
  (`_netdev` tells the system to wait for the network before mounting).

---

## 2. Navigating the Filesystem

### 2.1 Navigation commands

| Command | Definition |
|---|---|
| `pwd` | **P**rint **W**orking **D**irectory: shows where you are now |
| `cd <dir>` | **C**hange **D**irectory: move into `<dir>` |
| `cd /` | Go to the **root** directory (the top of the whole filesystem) |
| `cd ~` or `cd` | Go to your **home** directory (`~`, the tilde, means home, e.g. `/home/ubuntu`) |
| `cd ..` | Go **up one** level, to the parent directory |
| `cd ../..` | Go up **two** levels |
| `cd -` | Go back to the previous directory |
| `ls` | List files in the current directory |
| `ls -la` | List **all** files, including hidden ones (`-a`), in **long** format (`-l`): permissions, owner, size, date |
| `mkdir <dir>` | Make a new directory |
| `cmd1 && cmd2` | Run `cmd2` only if `cmd1` succeeded |

**Chaining example:** go home, make a folder and enter it, all in one line:
```bash
cd ~ && mkdir testdir && cd testdir
```

### 2.2 Relative paths: `.` and `..`

Starting from `/home/user/project`:

| Path | Goes to |
|---|---|
| `.` | `/home/user/project` (stay here) |
| `..` | `/home/user` (up 1) |
| `../..` | `/home` (up 2) |
| `../../..` | `/` (up 3) |

### 2.3 Important directories for DevOps

| Path | What lives there |
|---|---|
| `/` | Root of the filesystem; every other path starts here |
| `~` (`/home/<user>`) | Your home directory |
| `~/.ssh` | Your SSH keys and SSH config (`cd ~/.ssh`). The tilde makes this path work for any user |
| `/etc` | System configuration files |
| `/etc/fstab` | Filesystems to mount at boot |
| `/etc/systemd/system` | Where admins put **custom systemd service files**; a key location for DevOps |
| `/usr/bin` | Programs installed by the package manager (`apt`) |
| `/usr/local/bin` | Programs and **custom scripts you install yourself**. Preferred for custom scripts, because it doesn't conflict with package-managed files |
| `/var/log` | Log files |
| `/opt` | Optional / third-party software |

**Hidden files and directories** start with a dot (e.g. `.ssh`, `.bashrc`). They don't show in plain `ls`; use `ls -a` or `ls -la`. They usually hold user configuration.

---

## 3. Finding Files and Commands

### 3.1 Definitions

| Command | Definition |
|---|---|
| `which <cmd>` | Shows the full path of the program that runs when you type `<cmd>` (it searches your `$PATH`). Example: `which python3` |
| `type <cmd>` | Tells you what `<cmd>` is: an alias, a shell built-in, a function or a file on disk |
| `whereis <cmd>` | Finds a command's **binary, source and man page** locations. It only searches standard system folders |
| `whatis <cmd>` | Prints a one-line description of a command, taken from its man page. Example: `whatis vmstat` |
| `locate <name>` | Searches a pre-built **database** of file names. Very fast, but it can miss files created since the database was last updated |
| `sudo updatedb` | Rebuilds the `locate` database |
| `find <where> <tests>` | Searches the live filesystem. Slower than `locate`, but always up to date and very flexible |
| `man <cmd>` | Opens the full manual page for a command (press `q` to quit) |
| `info <cmd>` | Opens the GNU info documentation (often more detailed than `man`) |
| `<cmd> --help` | Prints a short usage summary of the options |

**Example: `which mandb`** asks *"Where is the `mandb` program, and is it installed?"* The output `/usr/bin/mandb` means yes, it's installed, and its program file lives at `/usr/bin/mandb`.

**Example: `whereis python`** lists every location related to Python (binaries, libraries, man pages). It is useful for seeing how a piece of software is spread across the filesystem and for spotting incomplete installations.

> **Best practice:** use `which` or `type` to find where a command lives before you go looking for it.

> **Why `whereis backup.sh` found nothing:** `whereis` only looks in standard binary and man-page folders. For your own scripts and files, use `find` or `locate`.

### 3.2 Installing `locate`

```bash
sudo apt update          # refresh the list of available packages
sudo apt upgrade         # upgrade installed packages to their newest versions
sudo apt install plocate # install the fast "locate" implementation
locate backup.sh
```

- **`apt update`**: Downloads the latest package lists. It does not install anything.
- **`apt upgrade`**: Installs newer versions of the packages you already have.
- **`apt install <pkg>`**: Installs a package.

### 3.3 `find`: options that matter

```bash
find <start-directory> [tests] [actions]
```

| Option | Meaning |
|---|---|
| `-name "file"` | Match by name (case-sensitive). Wildcards work: `-name "*.log"` |
| `-iname "file"` | Match by name, ignoring case |
| `-type f` | Only **files** |
| `-type d` | Only **directories** |
| `-print` | Print each match (this is the default action) |
| `2>/dev/null` | Throw away error messages such as "Permission denied" |
| `sudo find …` | Run as root so it can search protected folders |

**Examples**
```bash
find / -type d -name "project" 2>/dev/null     # find a directory named project, hide errors
sudo find / -type f -name "backup.sh"          # find every file named backup.sh
find /usr -name core -type f                   # files named "core" under /usr
sudo find /usr -name core -type d -print       # directories named "core" under /usr
```

**Sample output** of `sudo find / -type f -name "backup.sh"`:
```
/home/ubuntu/devops/scripts/backup.sh
/home/ubuntu/backup.sh
find: '/run/user/1000/gvfs': Permission denied
find: '/run/user/1000/doc': Permission denied
/opt/scripts/backup.sh
```
There are three copies of `backup.sh`. The "Permission denied" lines come from special desktop (FUSE) mounts that even root can't read. They are harmless; hide them with `2>/dev/null`.

### 3.4 `find` mistakes from my history

| What I typed | Why it failed | Correct |
|---|---|---|
| `find / type -name "backup.sh"` | `type` without a dash is read as a *folder name* to search | `find / -type f -name "backup.sh"` |
| `find / type -f -name …` | Same problem, and `-f` is not a test | `find / -type f -name …` |
| `find / -type -f -name …` | `-type` takes a letter (`f`, `d`), not `-f` | `find / -type f -name …` |
| `find / -type  -name …` | `-type` is missing its letter | `find / -type f -name …` |
| `what pidstat` | No such command | `whatis pidstat` |
| `whatis ls -la` | `whatis` takes command names, not their options | `whatis ls` |

---

## 4. Shell Productivity

### 4.1 Command-line editing shortcuts

These are useful when commands get long: long paths, Docker or Kubernetes commands, scripts with many arguments.

| Shortcut | Action |
|---|---|
| `Ctrl + A` | Move to the **beginning** of the line |
| `Ctrl + E` | Move to the **end** of the line |
| `Ctrl + U` | Delete everything **before** the cursor |
| `Ctrl + K` | Delete everything **after** the cursor |
| `Ctrl + W` | Delete the previous **word** |
| `Ctrl + R` | Search back through your command history (extra) |
| `Ctrl + L` | Clear the screen, same as `clear` (extra) |
| `Ctrl + C` | Stop the running command (extra) |
| `Tab` | Auto-complete commands and file names (extra) |

### 4.2 Aliases

An **alias** is a shortcut name for a command or a combination of commands. Aliases are expanded before the command runs, so they are ideal for adding default options or giving long commands memorable names.

```bash
alias name='command'      # syntax
```

| Command | Definition |
|---|---|
| `alias ll='ls -la'` | Typing `ll` now runs `ls -la` |
| `alias k=kubectl` | `k get pods` runs `kubectl get pods` |
| `alias tf=terraform` | `tf plan` runs `terraform plan` |
| `alias gst='git status'` | `gst` runs `git status` |
| `alias` | Lists all current aliases |
| `unalias ll` | Removes the `ll` alias |

> An alias created at the prompt lasts **only for the current shell session**. To keep it, add the `alias …` line to `~/.bashrc`, then run `source ~/.bashrc`.

### 4.3 Short-lived vs long-running commands and background jobs

There are two kinds of commands:
1. **Short-lived** ones, which finish quickly (`ls`, `pwd`, `cat`).
2. **Long-running** ones, which keep going (tailing logs with `tail -f`, running a server or app).

Adding `&` at the end starts a command **in the background**, so you get your prompt back.

| Command | Definition |
|---|---|
| `sleep 30 &` | `sleep 30` waits 30 seconds; `&` runs it in the background |
| `jobs` | Lists background and stopped jobs in this shell |
| `fg` / `fg %1` | Brings a job back to the **f**ore**g**round |
| `bg` / `bg %1` | Resumes a stopped job in the **b**ack**g**round |
| `Ctrl + Z` | Pauses (stops) the foreground job so you can `bg` it |
| `kill %1` | Ends job number 1 |

### 4.4 History and history expansion

| Command | Definition |
|---|---|
| `history` | Lists the commands you've run, with numbers |
| `!!` | Re-runs the previous command (e.g. `sudo !!`) |
| `!123` | Re-runs command number 123 from `history` |
| `!$` | The **last** argument of the previous command |
| `!*` | **All** arguments of the previous command (everything except the command name) |
| `!:2*` | Arguments from word 2 to the end (word 0 = the command name) |
| `clear` | Clears the terminal screen |

**Careful with `!*`.** My notes gave the example `grep pattern file1 file2 file3`, then `sed -i 's/old/new/g' !*`. But `!*` brings back **every** argument, including `pattern`, so `sed` would also try to edit a file called `pattern`. The same applies to the Kubernetes example. Select only the words you need instead:

```bash
grep pattern file1 file2 file3
sed -i 's/old/new/g' !:2*          # -> file1 file2 file3

kubectl get pods pod1 pod2 pod3
kubectl delete pods !:3*           # -> pod1 pod2 pod3
```

---

## 5. Redirection, Pipes, tee & Exit Codes

### 5.1 File descriptors

| FD | Name | Meaning |
|---|---|---|
| `0` | **stdin** | Standard input (the keyboard, or a pipe) |
| `1` | **stdout** | Standard output (normal results) |
| `2` | **stderr** | Standard error (errors, warnings, diagnostics) |

Keeping normal output (fd 1) separate from errors (fd 2) is a core Unix idea: success and failure information can be handled independently.

### 5.2 Redirection operators

| Operator | Definition | Example |
|---|---|---|
| `>` | Send stdout to a file (**overwrites** it) | `echo "Line 1" > output.txt` |
| `>>` | Send stdout to a file (**appends** to it) | `echo "Line 2" >> output.txt && cat output.txt` |
| `2>` | Send stderr to a file (overwrites) | `ls /nonexistent 2> error.txt` |
| `2>>` | Send stderr to a file (appends) | `backup.sh 2>> /var/log/backup-errors.log` |
| `1> … 2> …` | Send output and errors to separate files | `command 1> output.txt 2> errors.txt` |
| `&>` or `> file 2>&1` | Send **both** stdout and stderr to the same file | `./deploy.sh &> deploy.log` |
| `2>/dev/null` | Throw errors away (`/dev/null` is a "black hole") | `find / -name x 2>/dev/null` |
| `<` | Read stdin from a file | `sort < names.txt` |

| Command | Definition |
|---|---|
| `echo "text"` | Prints text |
| `echo -e "a\nb"` | `-e` turns on escape sequences such as `\n` (new line) and `\t` (tab) |
| `cat <file>` | Prints the contents of a file |

**Why this matters in DevOps**
- Save errors from a failed deployment: `terraform apply 2> terraform-errors.log`
- Cron jobs have no terminal, so errors must go to a file: `backup.sh 2>> /var/log/backup-errors.log`
- When troubleshooting, split errors from output: `command 2> errors.txt 1> output.txt`

### 5.3 The pipe `|`

The pipe connects the **stdout of one command to the stdin of the next**. Small tools that each do one job are combined to solve bigger problems, with no temporary files. The commands in a pipeline run at the same time, in separate processes.

| Example | What it does |
|---|---|
| `docker ps \| grep web` | Finds containers with "web" in their details |
| `grep ERROR access.log` | Finds error lines in a log (simpler than `cat access.log \| grep ERROR`) |
| `kubectl get pods \| grep Pending` | Finds pods stuck in Pending |
| `echo -e "zebra\napple\nbanana" \| sort` | Sorts the three words alphabetically |

| Command | Definition |
|---|---|
| `grep <pattern>` | Prints only the lines that contain the pattern |
| `sort` | Sorts lines alphabetically (`-n` sorts numbers, `-r` reverses) |

### 5.4 `tee`: save **and** see output

`tee` reads stdin and writes it to the screen (stdout) **and** to one or more files at the same time, like a T-junction in plumbing. Unlike `>`, you still see the output and can keep piping it onwards.

| Command | Definition |
|---|---|
| `echo "Visible output" \| tee output.txt` | Shows the text **and** saves it to `output.txt` (overwrites) |
| `cmd \| tee -a log.txt` | `-a` **appends** instead of overwriting |
| `./deployment.sh \| tee deployment.log` | Watch a deployment's progress live while keeping a log |
| `echo "x" \| sudo tee /etc/somefile` | Write to a root-owned file (`sudo echo x > file` does **not** work, because the redirect isn't run as root) |

### 5.5 Exit codes

Every command returns an exit code when it finishes.

| Code | Meaning |
|---|---|
| `0` | Success |
| Non-zero (1–255) | Failure. The meaning of each number depends on the program |

| Command | Definition |
|---|---|
| `echo $?` | Prints the exit code of the **last** command |
| `cmd1 && cmd2` | Runs `cmd2` only if `cmd1` exited with 0 |
| `cmd1 \|\| cmd2` | Runs `cmd2` only if `cmd1` failed |

```bash
ls /nonexistent 2> error.txt; echo $?   # prints 2 (failure)
cat error.txt                           # ls: cannot access '/nonexistent': No such file or directory
```

---

## 6. System Monitoring & Performance

> `mpstat`, `iostat`, `pidstat` and `sar` come from the **sysstat** package: `sudo apt install sysstat`.
>
> Many of these tools use the pattern **`command <interval> <count>`**. For example, `vmstat 1 5` = report every **1** second, **5** times.

### 6.1 Definitions

| Command | Definition |
|---|---|
| `free -h` | Memory (RAM) and swap: total, used, free, cached |
| `vmstat` | **V**irtual **m**emory **stat**istics: processes, memory, swap, I/O, CPU in one line |
| `vmstat 1 5` | The same, every 1 second, 5 times |
| `mpstat 1 5` | **M**ulti-**p**rocessor stats: CPU usage (user, system, idle, iowait). Add `-P ALL` for each core |
| `iostat` | CPU and **disk I/O** stats: reads and writes per second for each device |
| `iostat 1` | Disk I/O every second, until `Ctrl+C` |
| `pidstat` | Per-**process** (PID) CPU usage |
| `pidstat 1 5` | Per-process stats every 1 second, 5 times |
| `sar` | **S**ystem **A**ctivity **R**eport: collects, reports and saves system activity (CPU, memory, network, disk, …) |
| `top` | Live, interactive view of processes sorted by resource use |
| `ps -ef` | Snapshot of **every** process (`-e`) in **full** format (`-f`): UID, PID, PPID, start time, command |
| `lsof` | **L**i**s**t **O**pen **F**iles. In Linux, network sockets are files too |
| `ss -tlnp` | Shows **s**ocket **s**tatistics: **T**CP, **l**istening, **n**umeric ports, with **p**rocess names |
| `uname` | Prints system information (`uname -a` = kernel version, hostname, architecture) |

### 6.2 `sar` options

| Command | Definition |
|---|---|
| `sar -u 2 5` | CPU utilisation every 2 seconds, 5 lines |
| `sar -r` | Memory usage |
| `sar -n DEV 1 5` | Network traffic per interface, every 1 second, 5 times |
| `sar -I --int=14 -o int14.file 2 10` | Stats for interrupt (IRQ) 14 every 2 s, 10 lines, saved to `int14.file` |
| `sar -r -n DEV -f /var/log/sysstat/sa16` | Memory and network stats from the saved daily file for the 16th |
| `sar -A` | **All** statistics in the current daily data file |
| `sar -m BAT 1 5` | Battery statistics (power management) |

> To keep history, data collection must be enabled: set `ENABLED="true"` in `/etc/default/sysstat`, then `sudo systemctl enable --now sysstat`. Daily files are saved in `/var/log/sysstat/saDD` (DD = day of the month).

**Reading `sar -n DEV 1 5` output** (my server, the Average line):
```
Average:        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
Average:           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
Average:       enp0s3      8.33     13.36      0.69      2.98      0.00      0.00      0.16      0.00
Average:      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

| Column | Meaning |
|---|---|
| `IFACE` | Network interface: `lo` = loopback, `enp0s3` = main network card, `docker0` = Docker bridge |
| `rxpck/s` / `txpck/s` | Packets **received** / **transmitted** per second |
| `rxkB/s` / `txkB/s` | Kilobytes received / transmitted per second |
| `rxcmp/s` / `txcmp/s` | Compressed packets per second |
| `rxmcst/s` | Multicast packets received per second |
| `%ifutil` | How much of the interface's bandwidth is in use |

**What it shows:** only `enp0s3` has traffic, and very little (under 3 kB/s). `lo` and `docker0` are idle.

**Reading `sar -m BAT 1 5`:** `%cap 95` = battery at 95% capacity; `cap/min +0.00` = no change per minute. On a VM this is the virtual battery reported by the host.

### 6.3 `top`: useful interactive keys

| Key | Action |
|---|---|
| `h` or `?` | Help screen |
| `q` / `Esc` | Quit / leave the help screen |
| `P` / `M` | Sort by CPU / by memory |
| `1` | Show each CPU core separately |
| `l`, `t`, `m` | Toggle the load-average, task/CPU and memory summary lines |
| `c` | Toggle between command name and full command line |
| `u` | Show only one user's processes |
| `V` | Forest (tree) view of parent and child processes |
| `H` | Show threads |
| `f` | Choose, order and sort the displayed fields |
| `<` / `>` | Move the sort column left / right |
| `k` | Kill a process (asks for the PID) |
| `r` | Renice: change a process's priority |
| `d` | Change the refresh delay |
| `L` / `&` | Find a string / find again |
| `Z`, `B`, `z`, `b` | Colours and bold display options |
| `W` | Save the current layout to the config file |

### 6.4 Counting processes per user (a pipeline)

```bash
ps -ef | awk '{print $1}' | sort | uniq -c
```

| Part | Definition |
|---|---|
| `ps -ef` | Lists every process; column 1 is the user (UID) |
| `awk '{print $1}'` | `awk` is a text-processing tool; this prints only the 1st column of each line |
| `sort` | Sorts the names so identical ones are next to each other (`uniq` needs this) |
| `uniq -c` | Collapses repeated lines into one and **counts** them (`-c`) |

**My output:** `root` had 115 processes and `ubuntu` 78. The rest are service accounts (`avahi`, `_chrony`, `syslog`, …).
- The `1 UID` line is just the **header** of `ps -ef` being counted.
- Names like `cups-br+` and `systemd+` are cut off at 8 characters by `ps`.
- A cleaner version, with no header and sorted by count:
  ```bash
  ps -eo user= | sort | uniq -c | sort -rn
  ```

### 6.5 `lsof -i -U`: open network and Unix sockets

| Option | Meaning |
|---|---|
| `-i` | Internet sockets (TCP/UDP, IPv4/IPv6) |
| `-U` | Unix domain sockets (local communication between processes on the same machine) |

**Output columns**

| Column | Meaning |
|---|---|
| `COMMAND` | Process name (cut short, e.g. `systemd-r` = systemd-resolved) |
| `PID` | Process ID |
| `USER` | Owner of the process |
| `FD` | File descriptor number + mode: `8u` = fd 8, opened for read **and** write (`r` = read, `w` = write) |
| `TYPE` | `unix`, `IPv4`, `IPv6`, … |
| `DEVICE`, `SIZE/OFF`, `NODE` | Kernel identifiers |
| `NAME` | The socket path or address, plus its state (`LISTEN`, `CONNECTED`, …) |

**Notable lines in my output** (summarised; the full output was about 735 lines):

| Process | Socket | Meaning |
|---|---|---|
| `sshd` (and `systemd`) | `TCP *:ssh (LISTEN)` IPv4 + IPv6 | SSH server listening on port 22, on all addresses |
| `systemd-resolved` | `_localdnsstub:domain` TCP/UDP | Local DNS resolver (127.0.0.53) |
| `avahi-daemon` | `UDP *:mdns` | mDNS / Bonjour local network discovery |
| `chronyd` | `UDP localhost:323` | Time synchronisation (NTP) control port |
| `containerd` | `TCP localhost:40173 (LISTEN)` | containerd's local API (used by Docker) |
| `NetworkManager` | `bootpc->…:bootps`, `dhcpv6-client` | DHCP client getting the IP address |
| `systemd` | `/run/docker.sock (LISTEN)` | The Docker API socket |

The warnings `can't stat() fuse… /run/user/1000/doc` are normal: they're desktop virtual filesystems that `lsof` can't read.

**Handy `lsof` variants**
```bash
sudo lsof -i :8080        # which process is using port 8080?
sudo lsof -i -P -n        # network sockets with numeric ports and IPs (faster)
lsof -p <PID>             # everything a process has open
lsof /path/to/file        # which process has this file open?
```

---

## 7. Troubleshooting Labs (Tasks)

### LAB 1 — "Project not found": find the missing directory

**Scenario:** You are on the right machine but in the **wrong directory**. Many support tickets start with "project not found" or "start.sh is missing". Often the file isn't missing at all; the user just isn't in the directory where it lives.

**Task:** A folder called `project` exists somewhere on the system. Find it and move into it before continuing.

**Always start by asking:**
1. Where am I?
2. What is in this directory?
3. Where could the project folder be?

**Solution steps**
```bash
pwd                                         # 1. Where am I?
ls -la                                      # 2. What is here?
find / -type d -name "project" 2>/dev/null  # 3. Where is "project"? (hide permission errors)
cd /path/shown/above/project                # move into it
pwd                                         # confirm
```

> **Takeaway:** When something is "missing", first confirm your current location.

---

### LAB 2 — `start.sh` exists but won't run

**Scenario:** `start.sh` is supposed to start the application, but it fails when you run it.

**Task:** Inspect the file, check its permissions and fix the specific problem. **Do not** replace the script with a new one; the goal is to understand **why** it doesn't run.

**Common reasons a script fails**
- You are running it from the wrong directory
- The file doesn't have **execute** permission
- The script points to a missing interpreter (a bad shebang line)
- The script starts another program that fails
- The command was typed with the wrong path

**Investigation steps**
```bash
ls -l start.sh            # check permissions; you want an x, e.g. -rwxr-xr-x
head -1 start.sh          # check the shebang, e.g. #!/bin/bash
cat start.sh              # read what the script actually does
./start.sh                # run it and READ the error message
ss -tlnp | grep 8080      # after the fix: is something listening on port 8080?
curl localhost:8080       # does the app respond?
```

| Command | Definition |
|---|---|
| `ls -l <file>` | Shows permissions, owner, size and date |
| `head -1 <file>` | Shows the first line of a file (`head -n 5` = first 5 lines) |
| `./start.sh` | Runs the script in the **current** directory (`./` = "here") |
| `chmod +x <file>` | Adds execute permission |
| `ss -tlnp \| grep 8080` | Checks whether a process is listening on TCP port 8080 |
| `curl localhost:8080` | Sends an HTTP request to the app to see whether it answers |

**Reading permissions: `-rwxr-xr-x`**
```
-    rwx    r-x    r-x
|    |      |      └── others: read, execute
|    |      └───────── group:  read, execute
|    └──────────────── owner:  read, write, execute
└───────────────────── file type (- = file, d = directory)
```

**Typical fixes (depending on what you find)**
| Symptom | Fix |
|---|---|
| `Permission denied` | `chmod +x start.sh` |
| `bad interpreter: No such file or directory` | Fix the first line, e.g. `#!/bin/bash` (check the path with `which bash`) |
| `No such file or directory` for a file *inside* the script | Run it from the right folder, or fix the path in the script |
| Script runs but the app isn't on port 8080 | Read the app's error output or log, and check what the script launches |

> **Takeaway:** A script needs both correct content **and** correct execute permissions.

---

### LAB 3 — Compare a broken log with a known-good log

**Scenario:** Logs show what a program observed while it was running. The **current** log tells you what is failing now; an **older working** log shows what the system looked like before the problem appeared.

**Task:** Compare the current failure log (`app.log`) with the older working log (`app.log.1`). Look for specific differences: **ports, file names, paths, error messages and success messages**. Don't only search for the word "ERROR"; also look for the last known good behaviour.

**Solution steps**
```bash
diff app.log app.log.1          # line-by-line differences
diff -u app.log.1 app.log       # "unified" view: easier to read (old first, new second)
grep -n -i "error\|fail" app.log      # error lines with line numbers
grep -n -i "listening\|started\|success" app.log.1   # what "good" looked like
```

| Command | Definition |
|---|---|
| `diff file1 file2` | Shows the lines that differ. `<` = only in file1, `>` = only in file2 |
| `diff -u old new` | Unified format: `-` = removed, `+` = added, with context lines |
| `grep -n` | Shows line numbers with matches |
| `grep -i` | Ignores upper/lower case |

> `app.log.1` is the naming used by **log rotation**: `.1` is the previous (older) log.

> **Takeaway:** Good debugging compares broken behaviour with known-good behaviour.

---

## 8. Practice Task Checklist

- [ ] **Navigation:** `cd /usr/local/bin`, then `cd ../..`, then `pwd`; you should be in `/usr`
- [ ] **Chaining:** `cd ~ && mkdir testdir && cd testdir`
- [ ] **Hidden files:** `cd ~/.ssh` and run `ls -la`
- [ ] **Aliases:** create `alias ll='ls -la'`, use it, then make it permanent in `~/.bashrc`
- [ ] **Background jobs:** run `sleep 30 &`, then `jobs`, then `fg`
- [ ] **Append:** `echo "Line 2" >> output.txt && cat output.txt`
- [ ] **Error redirection:** `ls /nonexistent 2> error.txt`, then `cat error.txt` (run them one at a time first)
- [ ] **Pipes:** run `echo -e "zebra\napple\nbanana"`, then `echo -e "zebra\napple\nbanana" | sort`
- [ ] **tee:** `echo "Visible output" | tee output.txt`, then `cat output.txt`
- [ ] **Exit codes:** run a good and a bad command, checking `echo $?` after each
- [ ] **Find:** `sudo find / -type f -name "backup.sh" 2>/dev/null`
- [ ] **Monitoring:** `vmstat 1 5`, `mpstat 1 5`, `iostat 1`, `pidstat 1 5`, `sar -n DEV 1 5`
- [ ] **Processes:** `ps -ef | awk '{print $1}' | sort | uniq -c`
- [ ] **LVM:** complete [Task 1.3](#13-task--build-an-lvm-volume-from-scratch) end to end, then grow the LV with `lvextend -r -L +100M`
- [ ] **Labs:** complete Labs 1, 2 and 3

---

## 9. Command Index (A–Z)

| Command | One-line definition |
|---|---|
| `alias` | Create a shortcut name for a command |
| `apt install / update / upgrade` | Install packages / refresh package lists / upgrade installed packages |
| `awk` | Text-processing tool; `awk '{print $1}'` prints column 1 |
| `bg` | Resume a stopped job in the background |
| `cat` | Print a file's contents |
| `cd` | Change directory |
| `chmod +x` | Make a file executable |
| `clear` | Clear the terminal screen |
| `curl` | Send HTTP requests from the terminal |
| `df -h` | Disk space of mounted filesystems |
| `diff` | Compare two files line by line |
| `echo` | Print text (`-e` enables `\n`, `\t`) |
| `fg` | Bring a job to the foreground |
| `find` | Search the filesystem live by name, type, size, … |
| `free -h` | RAM and swap usage |
| `grep` | Print lines that match a pattern |
| `head` | Show the first lines of a file |
| `history` | List previously run commands |
| `info` | GNU info documentation |
| `iostat` | CPU and disk I/O statistics |
| `jobs` | List background and stopped jobs |
| `locate` | Fast file-name search using a database |
| `ls` / `ls -la` | List files / list all files in long format |
| `lsblk` | Tree of disks, partitions and LVs |
| `lsof` | List open files and sockets |
| `lvcreate` / `lvextend` / `lvdisplay` | Create / grow / show Logical Volumes |
| `man` | Manual page for a command |
| `mkdir` | Create a directory |
| `mkfs.xfs` | Format a device with XFS |
| `mount` / `mount -a` | Attach a filesystem / mount everything in `/etc/fstab` |
| `mpstat` | Per-CPU usage statistics |
| `parted` | Create and edit partitions |
| `pidstat` | Per-process statistics |
| `ps -ef` | Snapshot of all processes |
| `pvcreate` / `pvdisplay` / `pvmove` | Create / show / move data off Physical Volumes |
| `pwd` | Print the current directory |
| `sar` | System activity reports (CPU, memory, network, …) |
| `sed -i 's/old/new/g'` | Find-and-replace text inside files, in place |
| `showmount -e` | List an NFS server's exports |
| `sleep` | Wait for a number of seconds |
| `sort` | Sort lines |
| `ss -tlnp` | Listening TCP ports and their processes |
| `tee` | Write output to the screen and to a file |
| `top` | Live process monitor |
| `type` | Show whether a name is an alias, built-in or file |
| `umount` | Detach a filesystem |
| `uname` | System and kernel info |
| `uniq -c` | Collapse duplicate lines and count them |
| `vgcreate` / `vgextend` / `vgdisplay` | Create / grow / show Volume Groups |
| `vmstat` | Memory, process, I/O and CPU summary |
| `whatis` | One-line description of a command |
| `whereis` | Locations of a command's binary, source and man page |
| `which` | Full path of the command that will run |
| `xfs_growfs` | Grow a mounted XFS filesystem |