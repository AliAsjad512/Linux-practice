# 7. Process Management - Monitoring & Control
Master Linux process management fundamentals. Learn to monitor running processes, understand process states, manage foreground and background jobs, control process priorities, and terminate processes safely. Essential skills for system administration and troubleshooting in production environments.
## Lab Summary
- Difficulty: intermediate
- Estimated time: 45 minutes
- Tasks: 27
- Validation checks: 0
- Interview questions: 6
- Topic: linux
- Skill Hubs: hub02-linux-management

---

## Theory Notes

Understanding Linux Processes

A running Linux system is full of processes.

Every command you run, every service in the background, every shell session, and every application is represented as a process. If you want to understand what a Linux machine is doing, you need to understand processes.

This lab teaches how to inspect, monitor, and control processes safely.

---

What Is A Process?

A process is a running program.

When you type a command, Linux starts a process for that command. That process gets a process ID, called a PID. It may use CPU, memory, files, network sockets, and other system resources while it runs.

Some processes finish quickly. Others keep running as services.

---

Why Process Visibility Matters

During troubleshooting, you often need to answer simple questions.

Is the service running? Which process is using CPU? Is a command stuck? What started this process? Is there more than one copy running? Which user owns it?

Process tools help answer these questions from the terminal.

---

Common Process Tools

`ps` shows process information. `top` gives an interactive view of resource usage. `pgrep` finds processes by name. `kill` sends signals to processes.

These tools are powerful because they let you inspect the system without guessing.

For example, if an application is not responding, you can check whether the process exists before assuming the network or configuration is broken.

---

Signals And Control

Linux controls processes using signals.

When you press `Ctrl+C`, the terminal sends an interrupt signal to the foreground process. When you use `kill`, you are also sending a signal.

Not every signal means "destroy immediately." Some ask the process to stop gracefully. Others force termination. A careful engineer starts with the least disruptive option and only escalates when necessary.

---

Foreground And Background Work

Some commands run in the foreground, which means the terminal waits for them to finish.

Other commands can run in the background. This is useful for long-running commands, but it also means you need to know how to list, resume, or stop jobs.

This is where shell job control connects to process management.

---

What You Will Practice

In this lab, you will list processes, inspect resource usage, find a process by name, send signals, and understand the difference between observing and controlling a process.

Takeaway:
Processes are the live activity of a Linux system. If you can inspect processes clearly, you can troubleshoot services with much more confidence.

---

## Task Checklist

- [ ] List Current Processes
- [ ] Install Process Tree
- [ ] View Process Tree
- [ ] Monitor Real-time Processes
- [ ] Find Specific Process by Name
- [ ] Show Process Details for Current Shell
- [ ] List Processes for Current User
- [ ] Check Process Resource Usage
- [ ] View Process Memory Usage
- [ ] Show Process State Information
- [ ] Find Parent-Child Process Relationships
- [ ] Show Process Start Time
- [ ] Find Zombie Processes
- [ ] Show Process Nice Values
- [ ] Show Process Command Arguments
- [ ] Install Process File Descriptors
- [ ] Show Process File Descriptors
- [ ] Check Process Network Connections
- [ ] Find Long-Running Processes
- [ ] Start a Long-Running Process
- [ ] Find Process ID
- [ ] Terminate the Process
- [ ] Run Process in Background
- [ ] List Background Jobs
- [ ] Bring Process to Foreground
- [ ] Change Process Priority
- [ ] Run Process with nohup

---

## Interview Questions

### 1. What is a process in Linux and how is it identified

A process is a running instance of a program, identified by a unique Process ID (PID) and associated with attributes like PPID, UID, GID, state, and resource usage

### 2. What is the difference between foreground and background processes and how do you manage them

Foreground processes run in the terminal and block it, while background processes run independently; they can be managed using &, jobs, bg, and fg commands

### 3. How do you find, monitor, and identify resource-heavy processes

You can use ps aux, top, or ps with sorting options like --sort=-%cpu or --sort=-%mem to identify processes consuming high CPU or memory

### 4. What is the difference between kill, kill -9, and pkill

kill sends a SIGTERM signal allowing graceful shutdown, kill -9 sends SIGKILL which forcefully terminates a process, and pkill kills processes by name instead of PID

### 5. What is a zombie process and how do you identify it

A zombie process is a terminated process whose parent has not collected its exit status; it can be identified using ps output where the process state shows Z

### 6. What is nice and renice and how do they affect process execution

Nice sets the priority of a process when it starts, and renice changes the priority of a running process; higher nice values lower priority and reduce CPU allocation while lower values increase priority


outline:
- init
- fork/exec
- ps
- kill
- fg/bg/jobs

start a process:
[command]

replace current process with another process:
exec [command]

see processes in your shell:
ps f

kill processes:
kill -15 [pid]     <- sends sigterm
kill     [pid]     <- also sends sigterm
kill -2  [pid]     <- sends sigint
kill -1  [pid]     <- sends sighup
kill -9  [pid]     <- sends sigkill

suspend a process running in foreground:



So you are seeing: “Which processes are listening on which ports for networking”

The netstat command with -tuln options shows network connections: 
't' for TCP, 
'u' for UDP, 
'l' for listening sockets, 
'n' for numeric addresses (no DNS lookup), and 
'p' for process information. 

This reveals which processes are listening on which ports, active connections, and network usage patterns. 

For example you will see something like : 

tcp  127.0.0.53:53   LISTEN   826/systemd-resolve

systemd-resolved meaning:
Running on 127.0.0.53 (localhost)
Port 53 (DNS)
Handles DNS queries locally, Only accessible from your machine, not external

tcp  0.0.0.0:22   LISTEN   899/sshd
sshd meaning:
Listening on port 22 (SSH)
0.0.0.0 = all network interfaces
Accepts connections from anywhere

This is how you connected to this machine
