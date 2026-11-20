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




This command shows processes with their elapsed time (etime). The etime format shows days-hours:minutes:seconds or hours:minutes:seconds. Long-running processes often include system daemons, database servers, and web servers. Identifying these helps understand system stability, find processes that might need restart, and monitor service uptime for maintenance planning.

ps -eo pid,etime,comm --sort=-etime | head -10


# Linux Process Management – Lab Solutions

> **Note:** Some commands show more detail with root. If you're a normal user and see missing information, put `sudo` in front.
> Replace `<PID>` with a real process ID (find one with `pgrep` or `ps`).

---

## 1. Display all currently running processes

```bash
ps aux
```

Or (different format, shows the parent PID):

```bash
ps -ef
```

**Why it matters:** This is the first command to run when a server is slow or acting strange. It shows everything running, who owns it, and how much CPU and memory it uses.

---

## 2. psmisc Installation

```bash
apt update && apt install -y psmisc
```

On RHEL:

```bash
dnf install -y psmisc
```

**Why it matters:** `psmisc` provides `pstree`, `killall`, and `fuser`. Minimal servers and containers often don't include it, so you install it when you need those tools.

---

## 3. Display processes in hierarchical tree format

```bash
pstree -p
```

`-p` shows each process's PID. Without `pstree`:

```bash
ps -ef --forest
```

**Why it matters:** The tree shows which process started which. It's useful for finding what launched a runaway process, or for seeing all the worker processes of a service like nginx.

---

## 4. Open an interactive process monitor

```bash
top
```

Inside `top`: press `P` to sort by CPU, `M` to sort by memory, `k` to kill a process, `q` to quit.
A friendlier version (if installed): `htop`

**Why it matters:** `top` updates live, so you can watch CPU and memory as they change. It's the go-to tool during an incident when you need to see what's eating resources right now.

---

## 5. Search for processes by name pattern

```bash
pgrep -a nginx
```

`-a` shows the full command too. Alternative:

```bash
ps aux | grep [n]ginx
```

The `[n]` trick stops `grep` from matching its own line.

**Why it matters:** You usually know the service name, not its PID. Use this to check whether a service is running before restarting it or troubleshooting it.

---

## 6. Display detailed information about your current shell process

```bash
ps -f -p $$
```

`$$` is a special variable that holds your current shell's PID.

**Why it matters:** It confirms which shell you're in and who its parent is. Handy when you've switched users with `su` or `sudo` and aren't sure which session you're in.

---

## 7. Show only processes owned by your user

```bash
ps -u $(whoami)
```

Any other user:

```bash
ps -u nginx
```

**Why it matters:** On a shared server, it shows what a specific user or service account is running. Useful for finding a user's stuck jobs or checking what an app account is doing.

---

## 8. Display processes sorted by CPU usage

```bash
ps aux --sort=-%cpu | head
```

`-` before `%cpu` means highest first. `head` shows the top 10.

**Why it matters:** When the server is slow and `top` shows high CPU, this tells you exactly which process is responsible. It's often the first step before deciding to restart or kill something.

---

## 9. Display processes sorted by memory consumption

```bash
ps aux --sort=-%mem | head
```

**Why it matters:** Use it when memory is running out or the server starts swapping. A process whose memory keeps growing over time may have a memory leak.

---

## 10. Display process states and status codes

```bash
ps -eo pid,stat,comm
```

Common `STAT` codes:

| Code | Meaning |
|---|---|
| `R` | Running |
| `S` | Sleeping (waiting for something; normal) |
| `D` | Uninterruptible sleep (usually waiting on disk or network I/O) |
| `T` | Stopped (e.g. paused with `Ctrl+Z`) |
| `Z` | Zombie (finished, but the parent hasn't cleaned it up) |
| `I` | Idle kernel thread |

Extra characters: `s` = session leader, `+` = foreground, `<` = high priority, `N` = low priority.

**Why it matters:** States explain *why* something is slow. Many processes stuck in `D`, for example, usually point to a disk or storage problem rather than a CPU problem.

---

## 11. Display process with parent process information

```bash
ps -eo pid,ppid,comm
```

`PPID` = the parent's PID.

**Why it matters:** If you kill a process and it keeps coming back, its parent (often a service manager like systemd) is restarting it. The PPID tells you what to stop instead.

---

## 12. Display when processes were started

```bash
ps -eo pid,lstart,comm
```

**Why it matters:** It tells you whether a service restarted recently. If an app started 5 minutes ago and you didn't restart it, it probably crashed and was restarted automatically.

---

## 13. Identify zombie processes in the system

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

No output means no zombies.

**Why it matters:** Zombies are processes that finished but weren't cleaned up by their parent. A few are harmless, but a growing number means a buggy parent process. The fix is usually to restart the **parent** (use the PPID column).

---

## 14. Display process priority and nice values

```bash
ps -eo pid,pri,ni,comm
```

`NI` (nice) ranges from **-20** (highest priority) to **19** (lowest priority). `0` is the default.

**Why it matters:** It shows which processes get CPU first. Use it to check whether a background job has been set to low priority, or whether something important is being starved of CPU.

---

## 15. Display full command lines with arguments

```bash
ps -eo pid,args
```

Or, without cutting off long lines:

```bash
ps auxww
```

**Why it matters:** Many processes share the same name (like `python3` or `java`). The full command line shows *which* script or config each one is running, so you can tell them apart.

---

## 16. Install Process File Descriptors

```bash
apt install -y lsof
```

On RHEL:

```bash
dnf install -y lsof
```

**Why it matters:** `lsof` ("list open files") isn't always installed. You need it to see which files, sockets, and ports a process has open.

---

## 17. Display open file descriptors for a process

```bash
lsof -p <PID>
```

Without `lsof`:

```bash
ls -l /proc/<PID>/fd
```

**Why it matters:** It shows which files a process is using. Two classic uses: finding which process holds a deleted file that still takes disk space (`lsof +L1`), and finding why you can't unmount a disk (`lsof /data`).

---

## 18. Show network connections for all processes

```bash
sudo ss -tulnp
```

- `-t` TCP, `-u` UDP, `-l` listening, `-n` numbers instead of names, `-p` show the process

With `lsof`:

```bash
sudo lsof -i
```

**Why it matters:** It answers "which process is using port 8080?" Essential when a service fails to start because its port is already taken, or when checking what is exposed to the network.

---

## 19. Identify processes that have been running for a long time

```bash
ps -eo pid,etime,comm --sort=-etime | head
```

`ETIME` = elapsed time since start, shown as `[[dd-]hh:]mm:ss`.

**Why it matters:** It helps find forgotten jobs or stuck scripts that never finished. It also confirms how long a service has been up without a restart.

---

## 20. Start a process that runs for a long time in the background

```bash
sleep 3600 &
```

`&` runs it in the background. The shell prints the job number and PID, e.g. `[1] 4321`.

**Why it matters:** Lets you start long tasks (backups, downloads, scripts) and keep using the same terminal. `sleep` is a safe way to practice process commands.

---

## 21. Find the PID of the running sleep process

```bash
pgrep sleep
```

Or:

```bash
pidof sleep
```

**Why it matters:** Commands like `kill`, `renice`, and `lsof -p` need a PID. Finding it by name is faster and safer than reading through the whole `ps` list.

---

## 22. Terminate the running sleep process using its PID

```bash
kill <PID>
```

This sends `SIGTERM` (15), asking the process to stop cleanly. If it won't stop:

```bash
kill -9 <PID>
```

Verify:

```bash
pgrep sleep
```

No output means it's gone.

**Why it matters:** Used to stop hung or runaway processes. Always try plain `kill` first so the process can save data and clean up; use `kill -9` only as a last resort.

---

## 23. Start a process that runs in the background without blocking the terminal

```bash
sleep 600 &
```

If a command is already running in the foreground: press `Ctrl+Z` to pause it, then:

```bash
bg
```

**Why it matters:** You don't have to open a second terminal or wait for a long command to finish. `Ctrl+Z` then `bg` rescues you when you forgot the `&`.

---

## 24. Display all background jobs running in your current shell

```bash
jobs -l
```

`-l` also shows the PID.

**Why it matters:** It lists what you started in the background in **this** terminal, with job numbers like `%1`, `%2`. You need those numbers for `fg`, `bg`, and `kill %1`.

---

## 25. Bring a background job back to the foreground

```bash
fg %1
```

`%1` is the job number from `jobs`. Plain `fg` brings back the most recent job.

**Why it matters:** Use it when a background job needs input, or when you want to watch its output or stop it with `Ctrl+C`.

---

## 26. Modify the priority of an already running process

```bash
renice -n 10 -p <PID>
```

Check the change:

```bash
ps -o pid,ni,comm -p <PID>
```

- Normal users can only **raise** the nice value (lower priority).
- Lowering it (e.g. `-n -5`) requires `sudo`.

**Why it matters:** When a heavy job (backup, report, compression) slows down important services, you can lower its priority without stopping it. The job keeps running, just more politely.

---

## 27. Run a process that continues even after you close the terminal

```bash
nohup sleep 3600 > /dev/null 2>&1 &
```

- `nohup` ignores the hang-up signal (`SIGHUP`) sent when the terminal closes
- `> /dev/null 2>&1` discards output (otherwise it goes to `nohup.out`)
- `&` runs it in the background

If it's already running in the background:

```bash
disown %1
```

**Why it matters:** Your SSH connection can drop at any time. For long jobs on a remote server (migrations, big copies, scripts), `nohup` makes sure the job keeps running after you log out. For real services, use a systemd service instead.
