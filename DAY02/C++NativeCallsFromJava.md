# Linux Native System Calls Used by Oracle C++ Host Agents

Oracle products such as:

- Oracle Enterprise Manager (OEM) Agent
- Oracle Clusterware
- Oracle Database
- Oracle Access Manager Agents
- OCI Native Provisioning Agents

are implemented largely in C/C++ because they need direct access to the operating system.

Instead of using Java APIs, these native agents call Linux system calls and kernel interfaces directly.

---

# 1. `/proc` Filesystem

## What is it?

`/proc` is a **virtual filesystem** created by the Linux kernel.

It doesn't store actual files on disk.

Instead, it exposes **live kernel information**.

```
Application
      │
      ▼
Read /proc/*
      │
      ▼
Linux Kernel
      │
      ▼
Current CPU/Memory/Processes
```

Oracle Host Agents constantly read `/proc` to monitor servers.

---

## Example 1 – Read CPU Information

Suppose Enterprise Manager wants to display:

```
CPU Model
CPU Cores
CPU Speed
```

### C++ Code

```cpp
#include <iostream>
#include <fstream>
#include <string>

int main()
{
    std::ifstream cpu("/proc/cpuinfo");

    std::string line;

    while(std::getline(cpu,line))
    {
        if(line.find("model name") != std::string::npos)
            std::cout << line << std::endl;
    }
}
```

Output

```
model name : Intel Xeon Platinum
```

### Oracle Usage

OEM Dashboard

```
Host Summary

CPU

Intel Xeon

16 Cores
```

---

## Example 2 – Read Memory Usage

```cpp
#include <fstream>
#include <iostream>
#include <string>

int main()
{
    std::ifstream mem("/proc/meminfo");

    std::string line;

    while(std::getline(mem,line))
    {
        if(line.find("MemTotal") != std::string::npos ||
           line.find("MemFree") != std::string::npos)
        {
            std::cout<<line<<std::endl;
        }
    }
}
```

Output

```
MemTotal: 32768000 kB
MemFree : 12456000 kB
```

OEM converts this into

```
Memory Used = 62%
```

---

# 2. `stat()`

## Purpose

`stat()` returns metadata about a file.

```
Application

↓

stat()

↓

Kernel

↓

inode
permissions
owner
size
timestamps
```

Oracle uses it before copying database files or backups.

---

## Example 1 – File Size

```cpp
#include <sys/stat.h>
#include <iostream>

int main()
{
    struct stat info;

    stat("data.db",&info);

    std::cout<<"Size = "
             <<info.st_size
             <<" bytes";
}
```

Output

```
Size = 104857600 bytes
```

### Oracle Example

Before cloning a database:

```
Check Datafile Size

↓

Enough Disk?

↓

Proceed
```

---

## Example 2 – Last Modified Time

```cpp
#include <sys/stat.h>
#include <ctime>
#include <iostream>

int main()
{
    struct stat info;

    stat("listener.ora",&info);

    std::cout<<ctime(&info.st_mtime);
}
```

Oracle Agent can detect

```
listener.ora changed

↓

Reload Listener
```

---

# 3. `fork()`

## Purpose

Creates a child process.

```
Parent

↓

fork()

↓

Parent + Child
```

The child is an almost identical copy of the parent.

Oracle uses it to launch installers, scripts, and database utilities without stopping the main agent.

---

## Example 1 – Launch Child Process

```cpp
#include <unistd.h>
#include <iostream>

int main()
{
    pid_t pid = fork();

    if(pid==0)
        std::cout<<"Child Process\n";
    else
        std::cout<<"Parent Process\n";
}
```

Possible output

```
Parent Process
Child Process
```

---

## Example 2 – Run Backup

```cpp
pid_t pid = fork();

if(pid==0)
{
    execl("/bin/tar",
          "tar",
          "-czf",
          "backup.tar.gz",
          "/u01/oradata",
          NULL);
}
```

Oracle Example

```
OEM Agent

↓

fork()

↓

Backup Process

↓

Main Agent Continues Running
```

---

# 4. `kill()`

Despite its name, `kill()` sends **signals**, not only termination requests.

Oracle uses it to stop or control database processes.

---

## Example 1 – Stop Process

```cpp
#include <signal.h>
#include <unistd.h>

int main()
{
    kill(3456,SIGTERM);
}
```

Meaning

```
Tell process 3456

↓

Shutdown Gracefully
```

---

## Example 2 – Reload Process

```cpp
kill(pid,SIGHUP);
```

Common use

```
listener.ora changed

↓

Send SIGHUP

↓

Listener Reloads Configuration
```

No restart needed.

---

# 5. `pthread`

Oracle Database is highly multithreaded.

Each client request may be handled by different worker threads.

---

## Example 1 – Create Thread

```cpp
#include <pthread.h>
#include <iostream>

void* work(void*)
{
    std::cout<<"Thread Running\n";
    return nullptr;
}

int main()
{
    pthread_t t;

    pthread_create(&t,
                   nullptr,
                   work,
                   nullptr);

    pthread_join(t,nullptr);
}
```

Output

```
Thread Running
```

Oracle Example

```
Host Agent

↓

Thread 1

CPU

Thread 2

Memory

Thread 3

Disk
```

All collected simultaneously.

---

## Example 2 – Thread Synchronization

```cpp
#include <pthread.h>

int counter = 0;

pthread_mutex_t lock;

void* worker(void*)
{
    pthread_mutex_lock(&lock);

    counter++;

    pthread_mutex_unlock(&lock);

    return nullptr;
}
```

Why?

Without the mutex:

```
Thread A

counter++

Thread B

counter++

Wrong Value
```

Oracle uses mutexes extensively for:

- Shared caches
- Session tables
- Monitoring data
- Connection pools

---

# 6. `socket`

Sockets allow two processes on different machines (or the same machine) to communicate over a network.

```
Java OEM Server

↓

TCP Socket

↓

C++ Host Agent
```

---

## Example 1 – TCP Client

```cpp
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main()
{
    int sock = socket(AF_INET,
                      SOCK_STREAM,
                      0);
}
```

This creates a TCP socket.

Oracle Agent later connects to

```
OEM Server

or

OCI Endpoint
```

---

## Example 2 – Send Monitoring Data

```cpp
const char* msg =
"CPU=25,MEM=61";

send(sock,
     msg,
     strlen(msg),
     0);
```

Java Server receives

```
CPU=25

Memory=61%

Disk=40%
```

and updates dashboards.

---

# Putting It All Together: Oracle OEM Host Agent

When an administrator opens the Enterprise Manager dashboard, the Java server requests host metrics from the native C++ agent.

```text
Administrator
      │
      ▼
OEM Java Server
      │
      │  Request Host Metrics
      ▼
C++ Host Agent
      │
      ├── Read /proc/cpuinfo
      ├── Read /proc/meminfo
      ├── stat() database files
      ├── pthread → Collect metrics in parallel
      ├── fork() → Run diagnostics if needed
      ├── kill() → Restart failed processes
      └── socket() → Send results to Java server
      │
      ▼
Java Dashboard
      │
      ▼
CPU: 23%
Memory: 68%
Disk: 41%
Database: Running
```

## Interview Takeaway

A strong interview answer is:

> "Oracle's Java applications orchestrate monitoring and provisioning workflows, but the actual interaction with the Linux operating system is handled by native C++ agents. These agents read kernel information through `/proc`, inspect files with `stat()`, create background processes using `fork()`, manage processes with `kill()`, collect data concurrently using `pthread`, and communicate with Java servers over TCP sockets. This separation lets Java focus on business logic while C++ performs low-level, high-performance system operations."
