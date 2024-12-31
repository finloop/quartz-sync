---
title: "My debugging toolbox: strace"
date: 2024-08-03T10:00:00+02:00
publish: true
---
From time to time, my usual debugging tools: good ol `printf`, `logger.log`, `console.log`, `gdb`, `pdb` are not enough to solve the problem. This happens more than often when multiple processes/threads are involved or performance is an issue.

One such tool that I found really useful is `strace`.

![Linux Performance Observability Tools. Source: https://www.brendangregg.com/Perf/linux_observability_tools.png](https://www.brendangregg.com/Perf/linux_observability_tools.png)
<figcaption>Linux Performance Observability Tools. Source: https://www.brendangregg.com/Perf/linux_observability_tools.png</figcaption>

See `man strace` (great read)!.

TL;DR

- Traces all syscalls (aka how your program interacts with os)
- It's like `printf` but it automatically prints *everything* being read/written to any *file descriptor* (file, socket etc.)
- Can show execution stack trace after each system call (flag `-k`)
- and follow forks! (with `-f`)
- Automatically decodes pids / file descriptors (flags `--decode-pids=comm` / `--decode-fds=all`)
- Each line follows: `callname(**args) = result` [^1]
- **Slows down traced program** [^2], sometimes ~ 100 times

To showcase these features, I've created a toy program. I'm going to show you how to discover what's it doing.

## Debugging missing config files

Some programs are not very verbose. This one complains that it cannot open a file, but it doesn't specify which one.

```zsh
$ ./demo
Cannot open file.
```

Running `strace ./demo` will print all the relevant output. You can filter it by adding `-eread,write,open,openat,writev` (common syscalls).

```sh
$ strace -eread,write,open,openat,writev ./demo
openat(AT_FDCWD, "example.txt", O_RDONLY) = -1 ENOENT (No such file or directory)
write(1, "Cannot open file.\n", 18Cannot open file.
```

Program tries to open file `example.txt` but it doesn't exist!

## Debugging running program

Lets run the demo in background:

```sh
$ ./demo &
[1] 104794
```

And connect to it with strace (`-p [PID]`):

```sh
$ strace -eread,write,open,openat,writev -p 104794
strace: Process 104794 attached
writev(3, [{iov_base="2137\n2137\n2137\n2137\n2137\n2137\n21"..., iov_len=8190}, {iov_base="2137\n", iov_len=5}], 2) = 8195
```

So our program is writing `2137\n` to file descriptor 3. What file is it? Use `--decode-fds=all`! And bam it's writing to file: `demo.txt`

```sh
$ strace -eread,write,open,openat,writev --decode-fds=all -p 104794
strace: Process 104794 attached
writev(3</home/pk/projects/strace/demo.txt>, [{iov_base="2137\n2137\n2137\n2137\n2137\n2137\n21"..., iov_len=8190}, {iov_base="2137\n", iov_len=5}], 2) = 8195
```

If our binary has debug symbols you can event print preety stack traces with `-k`:

```sh
strace -eread,write,open,openat,writev  -p 104794 -k
strace: Process 104794 attached
writev(3, [{iov_base="2137\n2137\n2137\n2137\n2137\n2137\n21"..., iov_len=8190}, {iov_base="2137\n", iov_len=5}], 2) = 8195
 > /usr/lib/x86_64-linux-gnu/libc.so.6(writev+0x17) [0x11aa57]
...
 > /home/pk/projects/strace/demo(waitDemo()+0x85) [0x138e]
 > /home/pk/projects/strace/demo(main+0x12) [0x149e]
 > /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_init_first+0x90) [0x29d90]
 > /usr/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0x80) [0x29e40]
 > /home/pk/projects/strace/demo(_start+0x25) [0x1245]
```

## Example traced program

This program was used to showcase neat features of strace.

```c++
#include <iostream>
#include <fstream>
#include <unistd.h>
using namespace std;

void waitDemo() {
  ofstream file;
  file.open("./demo.txt");
  if (!file.is_open()) {
    exit(1);
  }
  while(true) {
    file << "2137\n";
    usleep(1000);
  }
}

void openFileDemo() {
  ifstream file;
  file.open ("example.txt");
  if (!file.is_open()) {
    cout << "Cannot open file.\n";
    exit(78);
  }
}

int main () {
  openFileDemo();
  waitDemo();
  return 0;
};

```

Compile with:

```sh
g++ main.cpp -g -o demo
```

[^1]: [strace(1) - Linux man page](https://linux.die.net/man/1/strace)
[^2]: [strace Wow Much Syscall, Gregg Brendan](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)

## More about strace

- [The magic of strace](http://web.archive.org/web/20150312221019/http://chadfowler.com/blog/2014/01/26/the-magic-of-strace/)
- [strace Wow Much Syscall, Gregg Brendan](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)
