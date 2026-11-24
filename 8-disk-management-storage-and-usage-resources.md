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
