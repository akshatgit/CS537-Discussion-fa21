# COMP SCI 537 Discussion Week 1

## CSL AFS Permission

CSL Linux machines use AFS (Andrew File System). It is a distributed filesystem (that why you see the same content when logging intodifferent CSL machines) with additional access control.

To see the permission of a directory

```shell
fs la /path/to/dir
```

You could also omit `/path/to/dir` to show the permission in the current working directory. You might get output like this:

```console
$ fs la /p/course/cs537-yuvraj/turnin
Access list for /p/course/cs537-yuvraj/turnin is
Normal rights:
  system:administrators rlidwka
  system:anyuser l
  asinha32 rlidwka
  yuvraj rlidwka
```

- `r` (read): allow to read the content of files
- `l` (lookup): allow to list what in a directory
- `i` (insert): allow to create files
- `d` (delete): allow to delete files
- `w` (write): allow to modify the content of a file or `chmod`
- `a` (administer): allow to change access control

Check [this](https://computing.cs.cmu.edu/help-support/afs-acls) out for more how to interpret this output.

You should test all your code in CSL.


## Intro to Xv6

To get the source code, copy xv6 tar file in your private directory:

```bash
$ cp /p/course/cs537-yuvraj/public/xv6.tar.gz .
$ tar -xvf xv6.tar.gz
```

To compile the xv6:

```bash
$ make
```

Recall an operating system is special software that manages all the hardware, so to run an OS, you need hardware resources like CPUs, memory, storage, etc. To do this, you could either have a real physical machine, or a virtualization software that "fakes" these hardwares. QEMU is an emulator that virtualizes some hardware to run xv6.

To compile the xv6 and run it in QEMU:

```bash
$ make qemu-nox
```

`-nox` here means no graphical interface, so it won't pop another window for xv6 console but directly show everything in your current terminal. The xv6 doesn't have a `shutdown` command; to quit from xv6, you need to kill the QEMU: press `ctrl-a` then `x`.

After compiling and running xv6 in QEMU, you could play walk around inside xv6 by `ls`.

```bash
$ make qemu-nox

iPXE (http://ipxe.org) 00:03.0 CA00 PCI2.10 PnP PMM+1FF8CA10+1FECCA10 CA00
                                                                            


Booting from Hard Disk..xv6...
cpu1: starting 1
cpu0: starting 0
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap8
init: starting sh
$ ls
.              1 1 512
..             1 1 512
README         2 2 2286
cat            2 3 16272
echo           2 4 15128
forktest       2 5 9432
grep           2 6 18492
init           2 7 15712
kill           2 8 15156
ln             2 9 15012
ls             2 10 17640
mkdir          2 11 15256
rm             2 12 15232
sh             2 13 27868
stressfs       2 14 16148
usertests      2 15 67252
wc             2 16 17008
zombie         2 17 14824
console        3 18 0
stressfs0      2 19 10240
stressfs1      2 20 10240
$ echo hello world
hello world
```

## Xv6 Syscalls
We will explore main files required to do p1A. 

- `Makefile`:
    -  `CPUS`: # CPUS for QEMU to virtualize (make it 1)
    -  `UPROGS`: what user-space program to build with the kernel. Note that xv6 is a very simple kernel that doesn't have a compiler, so every executable binary replies on the host Linux machine to build it along with the kernel.
    - Exercise: add a new user-space application called `hello` which prints `"hello world\n"` to the stdout.

- `usys.S`: where syscalls interfaces are defined in the assembly. This is the entry point to syscalls. 
    - Code for `usys.S` 
        ```assembly
        #include "syscall.h"
        #include "traps.h"

        #define SYSCALL(name) \
        .globl name; \
        name: \
            movl $SYS_ ## name, %eax; \
            int $T_SYSCALL; \
            ret

        SYSCALL(fork)
        SYSCALL(exit)
        SYSCALL(wait)
        # ...
        ```
    - It first declares a macro `SYSCALL`, which takes an argument `name` and expands it.

        ```assembly
        #define SYSCALL(name) \
        .globl name; \
        name: \
            movl $SYS_ ## name, %eax; \
            int $T_SYSCALL; \
            ret
        ```    
    - For e.g. let `name` be `getpid`, the macro above will be expanded into

        ```assembly
        .globl getpid
        getpid:
            movl $SYS_getpid, %eax
            int $T_SYSCALL
            ret
        ```
    - Where is `SYS_getpid` and `T_SYSCALL` located?
    - so it will be further expanded into
        ```assembly
        .globl getpid
        getpid:
            movl $11, %eax
            int $64
            ret
        ```
    - Run `gcc -S usys.S` to see all syscalls. 
        ```console
        $ gcc -S usys.S
        // ...
        .globl dup; dup: movl $10, %eax; int $64; ret
        .globl getpid; getpid: movl $11, %eax; int $64; ret
        .globl sbrk; sbrk: movl $12, %eax; int $64; ret
        // ...
        ```
    - `.globl getpid` declares a label `getpid` for compiler/linker; when someone in the user-space executes `call getpid`, the linker knows which line it should jump to. It then moves 11 into register `%eax` and issues an interrupt with an operand 64.
- `trapasm.S`: The user-space just issues an interrupt with an operand `64`. The CPU then starts to run the kernel's interrupt handler.
    - It builds what we call a trap frame, which basically saves some registers into the stack and makes them into a data structure `trapframe`. 

    - `trapframe` is defined in `x86.h`:

    ```C
    struct trapframe {
    // registers as pushed by pusha
    uint edi;
    uint esi;
    uint ebp;
    uint oesp;      // useless & ignored
    uint ebx;
    uint edx;
    uint ecx;
    uint eax;

    // rest of trap frame
    ushort gs;
    ushort padding1;
    ushort fs;
    ushort padding2;
    ushort es;
    ushort padding3;
    ushort ds;
    ushort padding4;
    uint trapno;

    // below here defined by x86 hardware
    uint err;
    uint eip;
    ushort cs;
    ushort padding5;
    uint eflags;

    // below here only when crossing rings, such as from user to kernel
    uint esp;
    ushort ss;
    ushort padding6;
    };
    ```
    - `alltraps` eventually executes `call trap`. The function `trap()` is defined in `trap.c`:


- `trap.c`: recall a syscall is implemented as an interrupt (sometimes also called "trap")
    - Trap code:
        ```C
        void
        trap(struct trapframe *tf)
        {
        if(tf->trapno == T_SYSCALL){
            if(myproc()->killed)
            exit();
            myproc()->tf = tf;
            syscall();
            if(myproc()->killed)
            exit();
            return;
        }
        // ...
        }
        ```
    - The 64 we just saw is `tf->trapno` here. `trap()` checks whether `trapno` is `T_SYSCALL`, if yes, it calls `syscall()` to handle it.

- `syscall.c` and `syscall.h`:

    - `void syscall(void)`: 
        ```C
        void
        syscall(void)
        {
        int num;
        struct proc *curproc = myproc();

        num = curproc->tf->eax;
        if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
            curproc->tf->eax = syscalls[num]();
        } else {
            cprintf("%d %s: unknown sys call %d\n",
                    curproc->pid, curproc->name, num);
            curproc->tf->eax = -1;
        }
        }
        ```
    - `Note:` See how to identify which syscall to call and where is the return value for completed syscalls. This is the heart of P1-A.


- `sysproc.c`:

    -  Kill syscall: Kill takes `PID` of the process to kill as an arguement, but this function has no arguement. How does this happen? 

        ```C
        int
        sys_kill(void)
        {
        int pid;

        if(argint(0, &pid) < 0)
            return -1;
        return kill(pid);
        }
        ```
        `Hint`: See `argint` is defined in `syscall.c`

- `proc.c` and `proc.h`:

    - `struct proc`: per-process state
    - `struct context`: saved registers
    - `ptable.proc`: a pool of `struct proc`
    - `static struct proc* allocproc(void)`: where a `struct proc` gets allocated from `ptable.proc` and **initialized**

- `user.h` and `defs.h`: what the difference compared to `user.h`? 

    `Hint:` [link](https://www.cs.virginia.edu/~cr4bd/4414/S2020/xv6.html)


## Shell Overview:
Shell is a program that takes input as "commands" line by line, parses them, then
- For most of the commands, the shell will create a new process and load it with an executable file to run. E.g. `gcc`, `ls`
- For some commands, the shell will react to them directly without creating a new process. These are called "built-in" commands. E.g. `alias`

If you are curious about which catalog a command falls into, try `which cmd_name`:

```console
# The output of the following commands may vary depending on what shell you are running
# Below is from zsh. If you are using bash, it may look different
$ which gcc
/usr/bin/gcc
$ which alias
alias: shell built-in command
$ which which
which: shell built-in command
```
### `fork`: So Is the Process Created

The syscall `fork()` creates a copy of the current process. We call the original process "parent" and the newly created process "child". But how are we going to tell which is which if they are the same??? The child process will get the return value 0 from `fork` while the parent will get the child's pid as the return value.

```C
pid_t pid = fork();
if (pid == 0) { // the child process will execute this
    printf("I am child with pid %d. I got return value from fork: %d\n", getpid(), pid);
    exit(0); // we exit here so the child process will not keep running
} else { // the parent process will execute this
    printf("I am parent with pid %d. I got return value from fork: %d\n", getpid(), pid);
}
```

You could find the code above in the repo as `fork_example.c`. After executing it, you will see output like this:

```console
$ ./fork_example
I am parent with pid 46565. I got return value from fork: 46566
I am child with pid 46566. I got return value from fork: 0
```
### `exec`: 

`fork` itself is not sufficient to run an operating system. It can only create a copy of the previous program, but we don't want the exact same program all the time. That's when `exec` shines. `exec` is actually a family of functions, including `execve`, `execl`, `execle`, `execlp`, `execv`, `execvp`, `execvP`... The key one is `execve` (which is the actual syscall) and the rest of them are just some wrappers in the glibc that does some work and then calls `execve`. For this project, `execv` is probably what you need. It is slightly less powerful than `execve`, but is enough for this project.

What `exec` does is: it accepts a path/filename and finds this file. If it is an executable file, it then destroys the current address space (including code, stack, heap, etc), loads in the new code, and starts executing the new code.

```C
int execv(const char *path, char *const argv[])
```

Here is how `execv` works: it takes a first argument `path` to specify where the executable file locates, and then provides the command-line arguments `argv`, which will eventually become what the target program receives in their main function `int main(int argc, char* argv[])`.

```C
pid_t pid = fork();
if (pid == 0) {
    // the child process will execute this
    char *const argv[3] = {
        "/bin/ls", // string literial is of type "const char*"
        "-l",
        NULL // it must have a NULL in the end of argv
    };
    int ret = execv(argv[0], argv);
    // if succeeds, execve should never return
    // if it returns, it must fail (e.g. cannot find the executable file)
    printf("Fails to execute %s\n", argv[0]);
    exit(1); 
}
// do parent's work
```

You could find the code above in the repo as `exec_example.c`. After executing it, you will see output exactly like executing `ls -l`.

As the last word of this section, I strongly recommend you to read the [document](https://linux.die.net/man/3/execv) yourself to understand how it works.

## `waitpid`: Wait for child to finish

Now you know the shell is the parent of all the processes it creates. The next question is, when executing a command, the shell should suspend and not asking for new inputs until the child process finishes executing.

```console
$ sleep 10 # this will create a new process executing /usr/bin/sleep
```

The command above will create a child process that does nothing other than sleeping for 10 seconds. During this period, you may notice your shell is also not printing a new prompt. This is what the shell does: it waits until the child process to terminate (no matter voluntarily or not). If you use `ctrl-c` to terminate `sleep`, you should see the shell would print the prompt immediately.

This is also be done by a syscall `waitpid`:

```C
pid_t waitpid(pid_t pid, int *status, int options);
```

This syscall will suspend the parent process and resume again when the child is done. It takes three arguments: `pid`, a pointer to an int `status`, and some flags `options`. `pid` is the pid of the child that the parent wants to waiit. `status` is a pointer pointing a piece of memory that `waitpid` could write the status to. `options` allows you to configure when `waitpid` should return. By default (`options = 0`), it only returns when the child terminates. This should be sufficient for this project.

Side note: NEVER write code like this:

```C
// assume we know pid
int* status;
waitpid(pid, status, 0); // QUIZ: what is wrong?
```

`*status` will be filled in the status that `waitpid` returns. This allows you to have more info about what happens to the child. You could use some macro defined in the library to check it, e.g. `WIFEXITED(status)` is true if the child terminated normally (i.e. not killed by signals); `WEXITSTATUS(status)` could tell you what is the exit code from the child process (e.g. if the child `return 0` from the main function, `WEXITSTATUS(status)` will give you zero).

Again, you should read the [document](https://linux.die.net/man/2/waitpid) yourself carefully.

Putting what we have discussed together, you should now have some idea of the skeleton of the shell

```C
pid_t pid = fork(); // create a new child process
if (pid == 0) { // this is child
    // do some preparation
    execv(path, argv);
    exit(1); // this means execv() fails
}
// parent
int status;
waitpid(pid, &status, 0); // wait the child to finish its work before keep going
// continue to handle the next command
```

Resources:

- Xv6 [Book](https://pdos.csail.mit.edu/6.828/2014/xv6/book-rev8.pdf)
- Man Pages - `exec` & `fork`