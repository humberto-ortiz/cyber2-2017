# Buffer overflows

## Vulnerable code

```
/* illustate buffer overflow */

int read_req(void) {
      char buf[128];		/* allocate buf on stack */
      int i;
      gets(buf);		/* unsafe! */
      i = atoi(buf);
      return i;
}

int main(void) {
    return read_req();
}

```

## Compiling the program

```
$ gcc -g -fno-stack-proctect read_req.c
```

## Examining the assembly

```
$ objdump -D a.out | grep 'read_req>:' -A 16
0000000000400566 <read_req>:
  400566:       55                      push   %rbp
  400567:       48 89 e5                mov    %rsp,%rbp
  40056a:       48 81 ec 90 00 00 00    sub    $0x90,%rsp
  400571:       48 8d 85 70 ff ff ff    lea    -0x90(%rbp),%rax
  400578:       48 89 c7                mov    %rax,%rdi
  40057b:       b8 00 00 00 00          mov    $0x0,%eax
  400580:       e8 bb fe ff ff          callq  400440 <gets@plt>
  400585:       48 8d 85 70 ff ff ff    lea    -0x90(%rbp),%rax
  40058c:       48 89 c7                mov    %rax,%rdi
  40058f:       b8 00 00 00 00          mov    $0x0,%eax
  400594:       e8 b7 fe ff ff          callq  400450 <atoi@plt>
  400599:       89 45 fc                mov    %eax,-0x4(%rbp)
  40059c:       8b 45 fc                mov    -0x4(%rbp),%eax
  40059f:       c9                      leaveq 
  4005a0:       c3                      retq   

```

## Preparing the input

```
$ python3 -c 'print("A" * 255)' > A.txt
```

## Running gdb

```
$ gdb a.out
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from a.out...(no debugging symbols found)...done.
(gdb) b read_req
Breakpoint 1 at 0x40056a
(gdb) r < A.txt
Starting program: /home/ubuntu/a.out < A.txt

Breakpoint 1, 0x000000000040056a in read_req ()
(gdb) 
(gdb) layout asm

   ┌───────────────────────────────────────────────────────────────────────────┐
B+>│0x40056a <read_req+4>   sub    $0x90,%rsp                                  │
   │0x400571 <read_req+11>  lea    -0x90(%rbp),%rax                            │
   │0x400578 <read_req+18>  mov    %rax,%rdi                                   │
   │0x40057b <read_req+21>  mov    $0x0,%eax                                   │
   │0x400580 <read_req+26>  callq  0x400440 <gets@plt>                         │
   │0x400585 <read_req+31>  lea    -0x90(%rbp),%rax                            │
   │0x40058c <read_req+38>  mov    %rax,%rdi                                   │
   │0x40058f <read_req+41>  mov    $0x0,%eax                                   │
   │0x400594 <read_req+46>  callq  0x400450 <atoi@plt>                         │
   │0x400599 <read_req+51>  mov    %eax,-0x4(%rbp)                             │
   │0x40059c <read_req+54>  mov    -0x4(%rbp),%eax                             │
   │0x40059f <read_req+57>  leaveq                                             │
   │0x4005a0 <read_req+58>  retq                                               │
   └───────────────────────────────────────────────────────────────────────────┘
native process 6846 In: read_req                             L??   PC: 0x40056a 
(gdb) 
```

The `info frame` command shows the stack frame before overflow:

```
(gdb) info frame
Stack level 0, frame at 0x7fffffffdfe0:
 rip = 0x400571 in read_req (read_req.c:6); saved rip = 0x4005aa
 called by frame at 0x7fffffffdff0
 source language c.
 Arglist at 0x7fffffffdfd0, args:
 Locals at 0x7fffffffdfd0, Previous frame's sp is 0x7fffffffdfe0
 Saved registers:
  rbp at 0x7fffffffdfd0, rip at 0x7fffffffdfd8
```
The info locals command shows the varibales local to this function.
```
(gdb) info locals
buf = '\000' <repeats 14 times>, "\377", '\000' <repeats 81 times>, "\001\000\00
0\000\000\000\000\000\375\005@", '\000' <repeats 20 times>
i = 0
```
Use the `nexti` instruction to single step the debugger until the
program crashes. You can see the stack frame with the `info frame`
command, and examine memory with the `x` comand:

```
(gdb) info frame
Stack level 0, frame at 0x7fffffffdfe0:
 rip = 0x400585 in read_req (read_req.c:7); saved rip = 0x4141414141414141
 called by frame at 0x7fffffffdfe8
 source language c.
 Arglist at 0x7fffffffdfd0, args:
 Locals at 0x7fffffffdfd0, Previous frame's sp is 0x7fffffffdfe0
 Saved registers:
 rbp at 0x7fffffffdfd0, rip at 0x7fffffffdfd8
(gdb) x/16w 0x7fffffffdfd0
0x7fffffffdfd0: 0x41414141      0x41414141      0x41414141      0x41414141
0x7fffffffdfe0: 0x41414141      0x41414141      0x41414141      0x41414141
0x7fffffffdff0: 0x41414141      0x41414141      0x41414141      0x41414141
0x7fffffffe000: 0x41414141      0x41414141      0x41414141      0x41414141
```
