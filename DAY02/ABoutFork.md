# Understanding `fork()` in Linux (C++)

`fork()` is one of the most fundamental Linux system calls. It **creates a new process** by duplicating the currently running process.

After a successful `fork()`, there are **two independent processes** running:

- **Parent Process** (the original process)
- **Child Process** (the newly created process)

Both processes continue executing **from the line immediately after `fork()`**.

---

# Your Code

```cpp
#include <unistd.h>
#include <iostream>

int main()
{
    pid_t pid = fork();

    if(pid == 0)
        std::cout << "Child Process\n";
    else
        std::cout << "Parent Process\n";
}
```

---

# Step-by-Step Execution

## Step 1

Initially, there is only one process.

```
main()
```

---

## Step 2

The process reaches:

```cpp
pid_t pid = fork();
```

Linux creates an exact copy of the process.

Before `fork()`:

```
        Process A
```

After `fork()`:

```
            fork()
               │
      ┌────────┴────────┐
      │                 │
Parent Process     Child Process
```

Now two processes are running independently.

---

## Step 3

Both continue from the next line:

```cpp
if(pid == 0)
```

But the value of `pid` is different in each process.

---

# Return Value of `fork()`

| Process | Return Value |
|----------|-------------:|
| Parent | Child's PID (> 0) |
| Child | 0 |
| Error | -1 |

Suppose the child PID is **5421**.

Then:

### Parent

```cpp
pid = 5421
```

### Child

```cpp
pid = 0
```

---

# Execution Flow

```
main()

    │

    ▼

pid = fork()

    │

 ┌──┴──────────────┐
 │                 │
 │                 │
 ▼                 ▼

Parent          Child

pid=5421        pid=0

 │                 │
 │                 │

else             if

 │                 │

 ▼                 ▼

Parent        Child
Process        Process
```

---

# Possible Output

```
Parent Process
Child Process
```

or

```
Child Process
Parent Process
```

The order is **not guaranteed** because the Linux scheduler decides which process runs first.

---

# Why are there two outputs?

After `fork()`, there are now **two processes**, each executing:

```cpp
std::cout << ...
```

Think of it as if the program were duplicated.

---

# Memory After `fork()`

Suppose before `fork()`:

```cpp
int x = 10;
```

Memory:

```
Original Process

x = 10
```

After `fork()`:

```
          fork()

      ┌──────────────┐
      │              │

 Parent          Child

 x=10            x=10
```

Each process gets its **own copy** of the variables.

Changing one does **not** affect the other.

---

# Example 1 – Parent and Child Modify Their Own Variables

```cpp
#include <unistd.h>
#include <iostream>

int main()
{
    int value = 100;

    pid_t pid = fork();

    if (pid == 0)
    {
        value += 50;
        std::cout << "Child value = " << value << '\n';
    }
    else
    {
        value -= 25;
        std::cout << "Parent value = " << value << '\n';
    }
}
```

### Possible Output

```
Parent value = 75
Child value = 150
```

---

## Explanation

Before `fork()`:

```
value = 100
```

After `fork()`:

```
              fork()

        ┌──────────────┐
        │              │

 Parent          Child

value=100      value=100
```

Parent changes its copy:

```
100 → 75
```

Child changes its copy:

```
100 → 150
```

Neither process changes the other's memory.

---

# Example 2 – Parent Waits for Child

Without waiting, the parent may finish before the child.

Use `wait()` so the parent waits until the child exits.

```cpp
#include <sys/wait.h>
#include <unistd.h>
#include <iostream>

int main()
{
    pid_t pid = fork();

    if (pid == 0)
    {
        std::cout << "Child is running\n";
    }
    else
    {
        wait(nullptr);

        std::cout << "Parent continues after child\n";
    }
}
```

### Output

```
Child is running
Parent continues after child
```

### Execution Timeline

```
Parent

fork()

 │

wait()

 │
 │
 │

Child

Runs

Prints

Exits

 │

Parent resumes

Prints
```

`wait(nullptr)` blocks the parent until the child terminates, ensuring a predictable order.

---

# Real-World Uses of `fork()`

`fork()` is widely used in Linux and Unix systems for:

- Creating new processes (e.g., shells launching commands)
- Web servers handling client requests
- Database server worker processes
- Background daemons
- Process isolation
- Build tools that execute external programs

Example:

```
Terminal

     ls

       │

       ▼

Shell

fork()

       │

Child Process

exec(ls)
```

The shell creates a child with `fork()`, then the child replaces itself with the `ls` program using `exec()`.

---

# Interview Summary

- `fork()` creates a **new child process** by duplicating the calling process.
- Both parent and child continue execution from the instruction immediately after `fork()`.
- The **parent receives the child's PID**, while the **child receives `0`**.
- Each process has its own virtual address space, so changes to variables in one process do not affect the other.
- Because both processes run concurrently, their execution order is determined by the operating system scheduler.
- `wait()` can be used when the parent needs to synchronize with the child before continuing.
