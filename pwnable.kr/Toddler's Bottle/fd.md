# Reverse Engineering Challenge: `fd`

## Challenge Description

```bash
fd@ubuntu:~$ ls
fd  fd.c  flag

fd@ubuntu:~$ cat fd.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]) {
    if(argc < 2){
        printf("pass argv[1] a number\n");
        return 0;
    }
    int fd = atoi(argv[1]) - 0x1234;
    int len = 0;
    len = read(fd, buf, 32);
    if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        setregid(getegid(), getegid());
        system("/bin/cat flag");
        exit(0);
    }
    printf("learn about Linux file IO\n");
    return 0;
}
```

## Analysis

### 1. Objective

The goal is to print the content of the `flag` file. Directly using `cat flag` from the shell is not allowed due to permission restrictions. We must find a way to trigger `system("/bin/cat flag")` through the `fd` program.

### 2. Source Code Behavior

The program expects a numeric argument `argv[1]`.

It computes:

```c
int fd = atoi(argv[1]) - 0x1234;
```

which means the given argument is subtracted by `0x1234` (4660 in decimal) to derive a file descriptor (`fd`).

Then it reads 32 bytes from that file descriptor into a buffer:

```c
len = read(fd, buf, 32);
```

If the buffer matches `"LETMEWIN\n"`, the flag is printed using:

```c
system("/bin/cat flag");
```

## Exploitation

### 1. Understanding File Descriptors

In Linux, the standard file descriptors are:

- `0`: stdin (input from keyboard)
- `1`: stdout (output to terminal)
- `2`: stderr (error output)

We want to read from `stdin`, which is file descriptor `0`. To make `fd = 0` in the code, we solve:

```
atoi(argv[1]) - 0x1234 = 0
→ argv[1] = 0x1234 = 4660
```

### 2. Exploit Steps

```bash
./fd 4660
LETMEWIN
```

After running the program, type `LETMEWIN` and press Enter.

If the input matches, the program prints:

```
good job :)
Mama! Now_I_understand_what_file_descriptors_are!
```

## Result

![result](/pwnable.kr/Toddler's%20Bottle/images/fd.png)

## Related Concepts

- File descriptors are integer handles to resources like files, sockets, and pipes in Unix-like systems.
- System calls like `read()`, `write()`, and `open()` use file descriptors for input/output operations.
- When using `fd = 0`, `read(fd, buf, 32)` reads input from the keyboard (stdin).
