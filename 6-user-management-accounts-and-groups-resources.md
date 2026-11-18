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
