# PennOS 💻
-   Adam Gorka @AdamEGorka, Andrew Lukashchuk @lukashchu, Hassan Rizwan @hrizwan3, Peter Proenca @peterbpro 
-   Last commit on 12/4/2023

## Introduction
PennOS is a Unix-like OS built in C, featuring a custom kernel with preemptive multitasking, virtual memory, a file system, and a command-line shell. It supports process scheduling, system calls, and inter-process communication, enabling efficient execution of concurrent processes. PennOS was built by four students as part of the final project for CIS 3800: Operating Systems (now CIS 5480) at the University of Pennsylvania. 

## Kernel Features
- **ucontext-based “threads”**  
  - Custom PCB struct storing `ucontext_t`, PID/PPID, children list, open FDs, priority, state, stack pointer, etc.  
  - `k_process_create()`, `k_process_kill()`, `k_process_cleanup()` manage kernel‑side process control  
- **User‑kernel syscall interface**  
  - `p_spawn()`, `p_waitpid()`, `p_kill()`, `p_exit()` as the only entry points for userland to request kernel actions  
- **Preemptive priority scheduler**  
  - Three priorities: -1 (interactive), 0 (normal), 1 (batch)  
  - Weighting: priority -1 runs 1.5× as often as 0, which runs 1.5× as often as 1  
  - 100 ms quanta via `setitimer(ITIMER_REAL)` + `SIGALRM` handler → `scheduler()`  
  - Idle process uses `sigsuspend()` to avoid 100% CPU when nothing runnable  
  - Round‑robin within each queue, starvation‑free by cyclical tick schedule  
  - **Logging** when scheduling a proess:  
    ```
    [ticks]    SCHEDULE    PID    PRIORITY    PROCESS_NAME
    ```  
  - **Screenshot:** ![Scheduler log exerpt showing scheduling](/img/scheduling.png)

- **Signals & process states**  
  - Defined signals: `S_SIGSTOP`, `S_SIGCONT`, `S_SIGTERM`  
  - States: `RUNNING`, `BLOCKED`, `STOPPED`, `ZOMBIE`  
  - Zombie / orphan handling with per‑PCB zombie queues and cleanup on parent exit  
  - `p_sleep(ticks)` to block a process until future tick; interruptible by `S_SIGTERM`  
  - `p_nice()` / `nice_pid()` to adjust priorities at runtime  

## File System Features
- **PennFAT (FAT16‑style) design**  
  - **FAT region**: fixed‐size, first entry stores block size (LSB) & FAT size (MSB)  
  - **Data region**: blocks numbered from 1, linked via 2‑byte FAT entries (0x0000 = free, 0xFFFF = end)  
  - **Root directory** only; 64‑byte directory entries storing name, size, firstBlock, type, perm, mtime  

- **Standalone `pennfat` utility**  
  - `mkfs FS_NAME BLOCKS_IN_FAT BLOCK_SIZE_CFG`  
  - `mount FS_NAME` / `umount`  
  - File commands: `touch`, `mv`, `rm`, `ls`, `cat` (with `-w`, `-a`), `cp` (with `-h`), `chmod`  
  - **Screenshot:** ![`pennfat` creating and mounting a new fs](/img/pennfat.png)

- **Kernel integration**  
  - User syscalls: `f_open()`, `f_read()`, `f_write()`, `f_lseek()`, `f_close()`, `f_unlink()`, `f_ls()`  
  - FD table in PCB maps to PennFAT or stdin/stdout as appropriate  
  - Abstract dispatch so host OS calls (e.g., read(2)) are never invoked by user programs  

## User Features
- **Built‑in programs** (spawned via `p_spawn()`):  
  - `cat`, `sleep`, `busy`, `echo`, `ls`, `touch`, `mv`, `cp`, `rm`, `chmod`, `ps`, `kill`, `zombify`, `orphanify`  
- **Shell‑only subroutines**:  
  - Job control: `nice`, `nice_pid`, `man`, `bg`, `fg`, `jobs`, `logout`  
- **Command parsing & redirection**  
  - `>`, `<`, `>>` support (no pipelines required)  
  - Background jobs with `&`  

- **Signal & terminal control**  
  - Host `SIGINT`/`SIGTSTP` trapped in shell → forwarded as `S_SIGTERM`/`S_SIGSTOP` to foreground job  
  - `terminal_owner_pid` logic: background processes trying to read stdin are stopped  

- **Screenshot:** ![Shell session demonstrating background jobs and fg/bg](/img/shell.png)

## Logging Facility
- Logs written to `./log` or user‑provided file via `./pennos fatfs [schedlog]`  
- Events:
  - **Scheduler**: `SCHEDULE`
  - **Lifecycle**: `CREATE`, `EXITED`, `SIGNALED`, `ZOMBIE`, `ORPHAN`, `WAITED`
  - **Priority**: `NICE`
  - **Blocking**: `BLOCKED` / `UNBLOCKED`
  - **Stop/Continue**: `STOPPED` / `CONTINUED`
- **Screenshot:** ![Excerpt of logging](/img/log.png)

## Error Handling
- Custom `ERRNO` variable (in `errno.h`) for all PennOS syscalls    
- Syscalls return `-1` & set `ERRNO`; shell calls `p_perror("msg")` to report errors  

## Directory Structure
```text
├── bin/                # User level function implementations: echo, sleep, etc.
├── doc/                # Documentation
├── log/                # System logging
├── src/                # Kernel level files including both PennFAT and PennOS systems
├── build/              # Compiled files
├── test/               # Unit & integration tests
├── Makefile
└── README.md
```
