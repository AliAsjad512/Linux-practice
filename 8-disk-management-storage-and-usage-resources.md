# 8. Disk Management - Storage and Usage
Master Linux disk and storage management fundamentals. Learn to monitor disk usage, understand filesystem concepts, manage storage capacity, and troubleshoot space-related issues. Essential skills for maintaining healthy systems and preventing storage-related outages in production environments.
## Lab Summary
- Difficulty: intermediate
- Estimated time: 45 minutes
- Tasks: 24
- Validation checks: 0
- Interview questions: 10
- Topic: linux
- Skill Hubs: hub02-linux-management

---

## Theory Notes

Disk Usage And Storage Awareness

Disk problems are common in real systems.

A server may look healthy but fail because a log file filled the disk, a backup consumed too much space, or an application wrote data into the wrong directory. When storage runs out, services can crash, databases can stop writing, and deployments can fail.

This lab teaches how to inspect disk usage before it becomes a bigger problem.

---

Disk Space Is Operational Evidence

Storage is not only an infrastructure detail.

It affects application behavior. If a filesystem is full, a program may fail to create files, write logs, save database records, or unpack deployments.

That is why engineers check disk usage during incidents and routine health checks.

---

Filesystem Usage

The `df` command shows filesystem-level disk usage.

It helps answer: how much space is used on each mounted filesystem? Which filesystem is close to full? Is the problem affecting the whole system or one mounted path?

The `-h` option makes sizes easier for humans to read.

---

Directory Usage

The `du` command helps inspect how much space files and directories use.

This is useful when `df` tells you a filesystem is full, but you still need to find what is consuming the space.

For example, logs, cache directories, backups, or temporary files may be larger than expected.

---

Inodes Matter Too

Disk space is not the only limit.

Linux filesystems also track inodes, which represent file entries. A system can have free space but still fail to create files if it runs out of inodes. This often happens when there are huge numbers of tiny files.

This is less common for beginners, but it is worth knowing early because it explains a confusing failure mode.

---

Safe Cleanup Thinking

Never delete files blindly just because disk usage is high.

First identify what is using space. Then decide whether the file is temporary, a log, a backup, application data, or something critical. Deleting the wrong file can make recovery harder or cause data loss.

Good cleanup is evidence-based.

---

What You Will Practice

In this lab, you will inspect filesystem usage, check directory sizes, find larger files, and build the habit of understanding storage before changing it.

Takeaway:
Disk checks are basic but important. A good engineer confirms what is using space before deciding what to clean up.

---

## Task Checklist

- [ ] Check Overall Disk Usage
- [ ] Show Specific Directory Size
- [ ] Find Largest Directories
- [ ] Check Filesystem Types
- [ ] Show Inode Usage
- [ ] List Block Devices
- [ ] Find Large Files
- [ ] Check /var Directory Usage
- [ ] Show Mounted Filesystems
- [ ] Check Specific Filesystem Usage
- [ ] Find Recently Modified Large Files
- [ ] Show Disk I/O Statistics
- [ ] Find Empty Directories
- [ ] Find Old Large Files
- [ ] Create a Virtual Disk To Understand Disk Partitioning
- [ ] Attach Virtual Disk
- [ ] Identify the New Disk
- [ ] Format the Disk
- [ ] Mount the Disk
- [ ] Verify Mounted Filesystem
- [ ] Simulate Continuous Log Writing
- [ ] Monitor Log File Growth
- [ ] Check Disk Usage of Partition
- [ ] Stop Log Writing Process

---

## Interview Questions

### 1. What is the difference between a disk, a partition, and a filesystem

A disk is the physical or virtual storage device, a partition is a logical division of that disk, and a filesystem is the structure created on a partition that allows data to be stored and organized (like ext4 or xfs)

### 2. What is the purpose of mounting a filesystem in Linux

Mounting attaches a filesystem to a directory so that its contents become accessible within the Linux directory tree, allowing users and applications to read and write data

### 3. How do you identify available disks and partitions in a system

You can use commands like lsblk, fdisk -l, or df -h to list disks, partitions, and mounted filesystems along with their sizes and mount points

### 4. What is the difference between df and du commands

df shows disk usage at the filesystem level (overall space used and available), while du shows disk usage at the file or directory level, helping identify which files consume space

### 5. What is inode exhaustion and how does it differ from disk space exhaustion

Inode exhaustion occurs when the filesystem runs out of available inodes (metadata entries for files), preventing new file creation even if disk space is free, whereas disk space exhaustion happens when storage capacity is fully used

### 6. Why is it recommended to use separate partitions for logs or application data

Separating logs or application data into dedicated partitions prevents them from filling up the root filesystem, ensuring system stability and isolating failures caused by excessive disk usage

### 7. What is a loop device and why is it used in disk management labs

A loop device allows a file to be treated as a block device, enabling safe simulation of disk operations like partitioning and formatting without modifying real disks

### 8. What happens if a filesystem is not mounted but exists on a partition

The data still exists on the partition but is not accessible through the filesystem hierarchy until it is mounted to a directory

### 9. How do you safely create and format a new partition in Linux

First identify the disk, create a partition using tools like fdisk, then format it using mkfs (e.g., mkfs.ext4), and finally mount it to a directory for use

### 10. How can you detect which directory is consuming the most disk space

You can use du -h with sorting (e.g., du -h | sort -h) or targeted checks like du -sh /path/* to identify directories consuming the most space


# Linux Disk & Storage Management – Lab Solutions

> **Note:** Many of these commands need root to see everything. If you're a normal user, put `sudo` in front.
> Add `2>/dev/null` to `find` and `du` commands to hide "Permission denied" noise.

---

## 1. Display filesystem usage for all mounted filesystems

```bash
df -h
```

`-h` = human-readable sizes (K, M, G).

**Why it matters:** This is the first command in any "disk full" incident. It shows every mounted filesystem with its size, used space, free space and `Use%`, so you instantly see which one is in trouble.

---

## 2. Calculate the total size of your home directory

```bash
du -sh ~
```

- `-s` = summary (one total, not every subfolder)
- `-h` = human-readable

**Why it matters:** `df` tells you a filesystem is full; `du` tells you *which folder* is using the space. Use it to check whether a user's home (old downloads, backups, logs) is the culprit.

---

## 3. Identify the largest directories in the root filesystem

```bash
sudo du -xh / --max-depth=1 2>/dev/null | sort -h | tail -10
```

- `-x` = stay on this filesystem (don't count other mounted disks)
- `--max-depth=1` = only top-level folders
- `sort -h | tail -10` = biggest 10, largest last

**Why it matters:** When `/` is filling up, this narrows the search from the whole system to one folder in seconds. Then you drill down into the biggest one and repeat.

---

## 4. Display filesystem types for all mounted filesystems

```bash
df -Th
```

`-T` adds a `Type` column (ext4, xfs, tmpfs, overlay…).

**Why it matters:** The filesystem type decides what you can do: XFS can grow but never shrink, ext4 can do both. Always check the type before resizing, repairing, or choosing a grow command.

---

## 5. Display inode usage for all filesystems

```bash
df -ih
```

**Why it matters:** A disk can show free space yet refuse new files because it ran out of inodes (one per file). This happens with millions of tiny files (cache, sessions, mail queues), and `df -h` alone won't reveal it.

---

## 6. Display all block storage devices and their hierarchy

```bash
lsblk
```

More detail (filesystem, UUID, mount point):

```bash
lsblk -f
```

**Why it matters:** Shows the full storage stack (disk → partition → LVM → mount point) in one tree. Run it before touching any disk so you never format or partition the wrong one.

---

## 7. Locate files larger than 100MB in the system

```bash
sudo find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

- `-xdev` = stay on the root filesystem
- `-size +100M` = bigger than 100 MB
- `-exec ls -lh {} \;` = show size and details of each file

**Why it matters:** Big forgotten files (old backups, ISO images, core dumps, huge logs) are the fastest space to reclaim. This finds them directly instead of guessing.

---

## 8. Examine disk usage breakdown for /var directory

```bash
sudo du -h /var --max-depth=1 2>/dev/null | sort -h
```

**Why it matters:** `/var` holds logs, caches, databases, and Docker data, so it's the most common place to fill up on servers. Usual suspects: `/var/log`, `/var/cache`, `/var/lib/docker`.

---

## 9. Display currently mounted filesystems and their options

```bash
findmnt
```

Plain list:

```bash
mount | column -t
```

**Why it matters:** Shows what is mounted where, and with which options (`ro`/`rw`, `noexec`, `nosuid`). Useful when a disk suddenly becomes read-only after errors, or to check security options on `/tmp`.

---

## 10. Display usage information for the root filesystem only

```bash
df -h /
```

**Why it matters:** When you only care about one filesystem, give `df` its path. It's ideal for quick checks, scripts, and monitoring alerts on `/`.

---

## 11. Locate large files modified in the last 7 days

```bash
sudo find / -xdev -type f -size +50M -mtime -7 -exec ls -lh {} \; 2>/dev/null
```

- `-mtime -7` = modified less than 7 days ago

**Why it matters:** If the disk filled up *recently*, the cause is usually a *new* big file: a runaway log, a fresh backup, or a dump. This finds exactly what grew this week.

---

## 12. Display disk input/output statistics for all devices

```bash
sudo apt install -y sysstat     # Ubuntu (RHEL: dnf install -y sysstat)
iostat -xz 2 3
```

- `-x` = extended stats, `-z` = hide idle devices
- `2 3` = every 2 seconds, 3 times

Watch `%util` (how busy the disk is) and `await` (wait time in ms).

**Why it matters:** When a server is slow but CPU looks fine, the disk is often the bottleneck. High `%util` and `await` confirm it; it's common with databases and heavy logging.

---

## 13. Locate empty directories that can be cleaned up

```bash
find /path/to/check -type d -empty
```

**Why it matters:** Old deployments and temporary jobs leave empty folders behind. They use little space, but cleaning them keeps directories tidy and speeds up tools that scan them.

> ⚠️ Never delete empty directories under system paths (`/`, `/etc`, `/var/lib`) blindly. Some apps expect them to exist.

---

## 14. Locate large files that haven't been accessed recently

```bash
sudo find / -xdev -type f -size +50M -atime +30 -exec ls -lh {} \; 2>/dev/null
```

- `-atime +30` = not read in more than 30 days

**Why it matters:** These are good candidates to archive, compress, or delete, because nobody is using them. Confirm with the owner first; some systems mount with `relatime`, so access times are approximate.

---

## 15. Create a file that will act as a virtual disk for partitioning

```bash
truncate -s 1G /root/virtual-disk.img
```

Or (writes real zeros, slower):

```bash
dd if=/dev/zero of=/root/virtual-disk.img bs=1M count=1024
```

**Why it matters:** A file-based disk lets you safely practise partitioning, formatting and mounting without touching a real disk. It's also how some test environments and CI pipelines simulate storage.

---

## 16. Attach the virtual disk file as a loop device so the system treats it like a real disk

```bash
sudo losetup -fP --show /root/virtual-disk.img
```

- `-f` = use the first free loop device
- `-P` = scan for partitions
- `--show` = print the device name (e.g. `/dev/loop18`)

**Why it matters:** A loop device makes a file behave like a block device (`/dev/loopX`), so `parted`, `mkfs` and `mount` work on it exactly as on a real disk. Snap packages and ISO images use the same mechanism.

---

## 17. Find the newly created loop device using lsblk

```bash
lsblk
losetup -a
```

`losetup -a` lists every loop device and which file it's attached to.

**Why it matters:** Always confirm the device name before formatting. Loop numbers vary between systems, and Snap packages already use many of them.

**Optional – create a partition on it:**

```bash
sudo parted /dev/loop18 --script mklabel gpt mkpart primary ext4 1MiB 100%
lsblk /dev/loop18       # shows loop18p1
```

---

## 18. Format the new partition with a filesystem

```bash
sudo mkfs.ext4 /dev/loop18p1      # if you created a partition
# or
sudo mkfs.ext4 /dev/loop18        # if you use the whole loop device
```

**Why it matters:** A disk or partition is raw space until it has a filesystem. `mkfs` creates that structure, and it **erases everything**, so double-check the device name first.

---

## 19. Mount the partition to a directory so it can be used

```bash
sudo mkdir -p /mnt/test_location
sudo mount /dev/loop18p1 /mnt/test_location
```

**Why it matters:** Mounting connects the filesystem to a folder, so files written there go to that disk. This is how servers put data, logs, or databases on separate disks.

---

## 20. Confirm that the partition is mounted correctly

```bash
df -h /mnt/test_location
findmnt /mnt/test_location
```

**Why it matters:** If a mount silently fails, files land on the root disk instead and fill it up. Always verify that the folder shows the *new* device and its size.

---

## 21. Simulate Continuous Log Writing

```bash
while true; do
    echo "INFO: app running at $(date)" >> /mnt/test_location/app.log
    sleep 1
done &
```

- `>>` = append a line each second
- `&` = run the loop in the background

**Why it matters:** This mimics a real application writing logs nonstop. Unmanaged log growth is one of the most common causes of full disks in production.

---

## 22. Observe how the log file grows over time in the mounted partition

```bash
tail -f /mnt/test_location/app.log
```

Press `Ctrl+C` to stop watching (this doesn't stop the loop).

File size:

```bash
ls -lh /mnt/test_location/app.log
```

**Why it matters:** `tail -f` is the everyday way to watch an application live while troubleshooting. You see new lines the moment they're written.

---

## 23. Verify that the mounted partition is being used as the log file grows

```bash
watch -n 2 df -h /mnt/test_location
```

Press `Ctrl+C` to exit.

**Why it matters:** It proves the writes go to the *mounted* disk, not to `/`. In production, the same check catches the moment a log partition starts filling, before it hits 100%.

---

## 24. Identify and stop the background process generating logs

Find it:

```bash
jobs -l
```

Or from any terminal:

```bash
ps aux | grep "[a]pp.log"
```

Stop it (use the job number or the PID of the **loop's `bash`**, not the `sleep`):

```bash
kill %1
# or
kill <PID>
```

Confirm the log stopped growing:

```bash
tail -n 1 /mnt/test_location/app.log; sleep 3; tail -n 1 /mnt/test_location/app.log
```

**Why it matters:** A runaway process will keep filling the disk until it's stopped; cleaning space without stopping it only buys minutes. Find it, stop it, then fix the root cause (log rotation, app config).

> ⚠️ Don't kill the `sleep` PID: the loop simply starts a new `sleep` a second later.

---

## Cleanup (after the lab)

```bash
sudo umount /mnt/test_location
sudo losetup -d /dev/loop18
rm /root/virtual-disk.img
```

Use your actual loop device name. Unmount first; detaching a mounted loop device fails.
