# 6. User Management - Accounts and Groups
Master Linux user and group management fundamentals. Learn to create, modify, and delete user accounts, manage groups, handle passwords, and understand user information files. Essential skills for system administration and multi-user environments in DevOps contexts.
## Lab Summary
- Difficulty: intermediate
- Estimated time: 45 minutes
- Tasks: 27
- Validation checks: 0
- Interview questions: 7
- Topic: linux
- Skill Hubs: hub02-linux-management

---

## Theory Notes

Users, Groups, And Access

Linux is built for more than one user.

Even on a small server, different people and services may need different levels of access. A developer may need access to project files. A web server process may need to read application files but not edit system configuration. An administrator may need temporary elevated privileges.

This lab teaches the basics of users and groups so you can understand who is allowed to do what.

---

Why User Management Matters

User management is a security skill, not just an administration task.

If everyone uses the same account, you lose accountability. If services run with too much privilege, a small application bug can become a system-wide problem. If users are placed in the wrong groups, they may gain access to files they should not touch.

Good Linux access control starts with clear users, clear groups, and careful permissions.

---

Types Of Users

Linux systems usually have regular users, system users, and the root user.

Regular users represent people who log in and work on the system. They usually have home directories under `/home`.

System users are often used by services. For example, a web server or database may run as its own user so it does not need full system access.

The root user has complete control of the system. Root can change almost anything, which is powerful but risky.

---

UIDs And GIDs

Linux identifies users and groups using numbers.

A user has a UID. A group has a GID. Names are easier for humans to read, but the system mainly works with these IDs.

UID `0` is root. Regular user IDs often start around `1000`, depending on the distribution. System users often use lower IDs.

Understanding IDs helps when you inspect files, containers, mounted volumes, and permissions.

---

Where User Information Lives

Linux stores user and group information in files.

`/etc/passwd` lists user accounts and basic details. `/etc/group` lists groups. `/etc/shadow` stores password-related data and is protected because it contains sensitive authentication information.

These files are not something to edit casually. In real work, you normally use commands such as `useradd`, `usermod`, `groupadd`, and `passwd` because they update the right files safely.

---

Groups Make Access Manageable

Groups let you manage access for sets of users.

Instead of giving permissions to each person one by one, you can place users into a group and give that group access to files or directories.

This is common in engineering teams. A deployment group may access release files. A support group may read logs. A service account may access only the application directory it needs.

---

What You Will Practice

In this lab, you will inspect users and groups, understand account files, create and modify users, work with groups, and see how Linux represents identity.

Takeaway:
Users and groups are how Linux turns identity into access control. Good systems give people and services only the access they need.

---

## Task Checklist

- [ ] Check Current User
- [ ] View User ID Information
- [ ] List All Users
- [ ] View Password File Format
- [ ] Check Root User Entry
- [ ] View Group Information
- [ ] Find User's Primary Group
- [ ] Check Groups You Belong To
- [ ] View Current User Details
- [ ] List Home Directories
- [ ] Check User Login Shell
- [ ] View User Description
- [ ] Check Last Login Information
- [ ] View Currently Logged Users
- [ ] Display User Account Aging
- [ ] Check Sudo Access
- [ ] View User Creation Defaults
- [ ] View Login History Summary
- [ ] Create a New User
- [ ] Set Password for User
- [ ] Switch to Another User
- [ ] Swtich back to root user
- [ ] Create a New Group
- [ ] Add User to Group
- [ ] Verify Group Membership
- [ ] Delete a User
- [ ] Delete a Group

---

## Interview Questions

### 1. What is the difference between whoami, who, and id

whoami shows the current user, who shows all logged-in users, and id shows UID, GID, and group memberships for a user

### 2. What is stored in /etc/passwd and /etc/shadow

/etc/passwd stores user account details like username, UID, home directory, and shell, while /etc/shadow stores encrypted passwords and is only accessible by root

### 3. What are UID and GID

UID uniquely identifies a user and GID identifies a group, with UID 0 reserved for the root user who has full privileges

### 4. What is the difference between a system user and a regular user

System users have UID less than 1000 and are used by services, while regular users have UID 1000 or higher and are meant for human interaction

### 5. What is the difference between usermod -G and usermod -aG

usermod -G replaces all existing group memberships, while usermod -aG appends the user to a new group without removing existing ones

### 6. Why is /etc/shadow not readable by normal users

It contains encrypted passwords and restricting access prevents unauthorized users from attempting password cracking

### 7. A user was added to a group but still cannot access resources, why

Group membership changes require a new session, so the user must log out and log back in or use newgrp for changes to take effect


# Linux Users and Groups – Outline

## Overview
- Normal and root users
- `/etc/passwd`
- `/etc/group`
- `/etc/shadow`

## Users
- Adding (`useradd`)
- Home directory & `/etc/skel`
- Removing (`userdel`)
- Modifying (`usermod`)
- `passwd`

## Groups
- Adding (`groupadd`)
- Modifying (`groupmod`)
- Removing (`groupdel`)

## Admin Related
- Root user
- sudoers
- `visudo`
- Account locking

grep $(whoami) /etc/passwd | cut -d: -f7


# Linux Users and Groups – Lab Solutions

> **Note:** Commands that change the system (creating or deleting users and groups) need root.
> If you're logged in as a normal user, put `sudo` in front. If you're already root (`#` prompt), you can drop `sudo`.

---

## 1. Display your current username

```bash
whoami
```

---

## 2. Display detailed user and group ID information

```bash
id
```

Shows your UID (user ID), GID (primary group ID), and all groups you belong to.

---

## 3. Display all user accounts on the system

```bash
cat /etc/passwd
```

Only the usernames:

```bash
cut -d: -f1 /etc/passwd
```

---

## 4. Examine the structure of the user account database

```bash
head -5 /etc/passwd
man 5 passwd
```

Each line has 7 fields separated by `:`

```
username:x:UID:GID:comment:home_directory:shell
root:x:0:0:root:/root:/bin/bash
```

| Field | Meaning |
|---|---|
| `username` | Login name |
| `x` | Password is stored in `/etc/shadow` |
| `UID` | User ID (0 = root) |
| `GID` | Primary group ID |
| `comment` | Full name or description |
| `home_directory` | Home folder |
| `shell` | Login shell |

---

## 5. Find the root user in the password file

```bash
grep '^root:' /etc/passwd
```

`^` means "at the start of the line," so it only matches the line that begins with `root:`.

---

## 6. Display all groups on the system

```bash
cat /etc/group
```

Only the group names:

```bash
cut -d: -f1 /etc/group
```

---

## 7. Determine which group is your primary group

```bash
id -gn
```

---

## 8. List all groups you are a member of

```bash
groups
```

Or:

```bash
id -Gn
```

---

## 9. Display your complete user account information

```bash
grep "^$(whoami):" /etc/passwd
```

Or:

```bash
getent passwd $(whoami)
```

---

## 10. View all user home directories

```bash
ls -l /home
```

Home directories of every account (field 6 of `/etc/passwd`):

```bash
cut -d: -f1,6 /etc/passwd
```

---

## 11. Find what shell you use for login

```bash
getent passwd $(whoami) | cut -d: -f7
```

Or:

```bash
echo $SHELL
```

---

## 12. Display your full name or description

```bash
getent passwd $(whoami) | cut -d: -f5
```

Field 5 is the comment (GECOS) field. It may be empty if no name was set.

---

## 13. View recent login information for your user

```bash
last $(whoami) | head
```

Most recent login per user (if available on your system):

```bash
lastlog -u $(whoami)
```

> **Why it matters:** Not so important in the lab setup, since lab terminals are ephemeral, but super critical when debugging production issues, especially when you need to know who logged in last.

The `last` command shows login history from the system logs. When followed by a username, it filters the output to show only that user's login sessions, including:

- Login time
- Logout time
- Session duration
- Terminal and IP address

It's useful for **security auditing**, **tracking user activity**, and **investigating unauthorized access attempts**.

---

## 14. Display all currently logged-in users

```bash
who
```

More detail (what each user is doing):

```bash
w
```

> **Why it matters:** Imagine your server is running slow and you want to know who else is using it, or you want to restart the system but first need to know whether any users are still connected. That's exactly what the `who` command is for.

The `who` command displays currently logged-in users with their terminal and login time. In multi-user environments, it shows active sessions and helps identify system usage. Each line shows:

- Username
- Terminal (like `tty1` for a local console, or `pts/0` for a remote SSH session)
- Login date and time
- Sometimes the remote host (IP address)

It's useful for **system monitoring** and **ensuring only authorized users are connected**.

---

## 15. Check password aging information for your account

```bash
sudo chage -l $(whoami)
```

Shows when the password was last changed, when it expires, and warning days.

---

## 16. Test if you can run commands with sudo

```bash
sudo -l
```

Lists what you're allowed to run with sudo. Quick test:

```bash
sudo whoami
```

If it prints `root`, sudo works for you.

---

## 17. Display default settings for new users

```bash
useradd -D
```

Related config files:

```bash
cat /etc/default/useradd
grep -v '^#' /etc/login.defs | grep -v '^$'
```

---

## 18. Display login failure attempts

```bash
sudo lastb
```

On RHEL (failed login lock counter):

```bash
sudo faillock --user $(whoami)
```

> **Why it matters:** The `lastb` command shows **bad** login attempts, which makes it helpful for security monitoring.

Failed logins can indicate:

- **Brute force attacks** (someone repeatedly guessing passwords)
- **Typos** (a user mistyping their password)
- **Account lockout situations**

`lastb` reads from `/var/log/btmp`, the file that logs failed authentication attempts. That's why it needs `sudo`: the file is readable only by root.

Regular monitoring of failed logins is important for **security auditing** and **incident response**.

---

## 19. Create a new user account named devuser with a home directory

```bash
sudo useradd -m devuser
```

`-m` creates the home directory `/home/devuser` (copied from `/etc/skel`).

Verify:

```bash
grep devuser /etc/passwd
ls -ld /home/devuser
```

---

## 20. Set a password for the newly created user devuser

```bash
sudo passwd devuser
```

Type the new password twice when asked.

---

## 21. Switch to the user devuser and load their full environment

```bash
su - devuser
```

The `-` loads devuser's full login environment (home directory, PATH, shell settings).

Check:

```bash
whoami
pwd
```

---

## 22. Switch back to root user

```bash
exit
```

This ends devuser's session and returns you to the previous user.

---

## 23. Create a new group named devops

```bash
sudo groupadd devops
```

Verify:

```bash
grep devops /etc/group
```

---

## 24. Add the user devuser to the devops group

```bash
sudo usermod -aG devops devuser
```

- `-a` = **append** (keep existing groups)
- `-G` = supplementary group

> ⚠️ Never forget `-a`. Without it, `-G` **replaces** all of the user's other supplementary groups.

---

## 25. Check which groups the user devuser belongs to

```bash
groups devuser
```

Or:

```bash
id devuser
```

---

## 26. Delete the user devuser and remove their home directory

```bash
sudo userdel -r devuser
```

`-r` also removes `/home/devuser` and the user's mail spool.

Verify:

```bash
grep devuser /etc/passwd
ls /home
```

---

## 27. Delete the group devops

```bash
sudo groupdel devops
```

Verify:

```bash
grep devops /etc/group
```

No output means the group is gone.


to remove user from a group
sudo gpasswd -d <username>