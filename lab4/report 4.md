# LAB 4 – Report

Zhuchkova Sofia FAF-241

## Purpose of the work

To use the xv6 shell side by side with bash and find out, by experiment, what a shell must do and what bash adds on top; to write a small program that prints the arguments it receives, which shows who expands `*` and removes quotes; and to read the source of the xv6 shell (`user/sh.c`).

## Part 1. Fix sleep for the current xv6

I checked which name my xv6 tree uses for the sleep system call:

```bash
cd ~/xv6-riscv
grep -n "pause\|sleep" user/user.h
```
Output:
```
25:int pause(int);
28:int sleep(int);
```

The tree declares `pause` (line 25), the new name that replaced `sleep()` in August 2025. My `user/sleep.c` from Lab 2 already calls `pause(atoi(argv[1]))`, so no change was needed. `sleep` is also already in `UPROGS` in the Makefile (line 154: `$U/_sleep\`).

I built and booted xv6 with `make qemu` and tested the program:

```
$ sleep 20
$ sleep
usage: sleep <ticks>
```

`sleep 20` waits 20 ticks and returns to the prompt. Without an argument it prints the usage message (to stderr) and exits with code 1.

## Part 2. bash versus the xv6 shell

I ran every command twice: in bash on Linux (in `~/shtest`, a folder with a file named `README`) and at the xv6 `$` prompt.

<p align="center"><img src="screenshots/fig2_1.png" width="600"><br><em>Figure 2.1 – The commands run in bash</em></p>

<p align="center"><img src="screenshots/fig2_2.png" width="600"><br><em>Figure 2.2 – The same commands run in xv6</em></p>

| Command | Tests | bash | xv6 | Why |
|---|---|---|---|---|
| `echo hello > f ; cat < f` | redirection | `hello` | `hello` | Both shells support `>` and `<`. |
| `ls \| grep READ` | pipe | `README` | `README         2 2 2441` | Both support pipes. The xv6 `ls` prints name, type, inode number and size, so its output looks different. |
| `(echo a; echo b) \| wc` | grouping, list | `2 2 4` | `2 2 4` | Both support `( )` grouping and `;`. |
| `echo one ; echo two` | sequence | `one` `two` | `one` `two` | Both run commands one after another. |
| `echo bg &` | background | `[1] 6377`, `bg`, `[1]+ Done echo bg` | `bg` (the prompt `$` came back first) | Both run the command in the background. bash adds job control (job number and a `Done` message), xv6 doesn't. |
| `ls *` | globbing | `README  f` | `ls: cannot open *` | bash expands `*` into file names. The xv6 shell does not, so `ls` received a literal `*`. |
| `echo $HOME` | variables | `/home/vboxuser` | `$HOME` | xv6 has no variables, the text is passed as it is. |
| `echo "a b"` | quotes | `a b` | `"a b"` | bash removes the quotes. xv6 keeps them as part of the text. |
| `mkdir d ; cd d ; ls ..` | PATH | `README  d  f` | `exec cd failed`, then the listing of the root directory | In bash `cd` works anywhere. In xv6 it only works at the start of a line, so inside a `;` list the shell looks for a program called `cd`, which doesn't exist, and `ls ..` then ran in the root directory. |
| arrow up, Tab, Ctrl+R | history, completion | all work | none work | xv6 only reads typed characters, so these keys do nothing useful. |

### Observe

**1. List the features both shells have, and the ones only bash has.**

Both have: running programs by name, redirection (`<`, `>`), pipes (`|`), sequences (`;`), background jobs (`&`) and grouping with `( )`.

Only bash has: wildcards (`*`), variables (`$HOME`), quote removal, command history (arrow up, Ctrl+R), Tab completion, a `PATH` to search for programs, `cd` inside a command list, and job control messages.

**2. In xv6, `ls` stops working after `cd d`. Why? How can I run `ls` from inside `d` anyway?**

xv6 has no `PATH`. When I type `ls`, the shell looks for a file named `ls` in the current directory. All the xv6 programs live in the root directory `/`. After `cd d` the current directory is `d`, which is empty, so the shell can't find `ls`. The fix is to type the absolute path `/ls`.

<p align="center"><img src="screenshots/fig2_3.png" width="600"><br><em>Figure 2.3 – ls failing after cd d, and the working alternative</em></p>

From my run:

```
$ cd d
$ ls
exec ls failed
$ /ls
.              1 27 32
..             1 1 1024
$ /ls ..
(the root files: README, cat, echo, grep, ls, ...)
```

**3. Which missing feature would you miss most in daily work, and which would be easiest to add?**

I would miss command history and Tab completion most, because in daily work I retype and correct long commands all the time. The easiest feature to add would be a simple `PATH`: when `exec` fails, the shell could try again with `/` in front of the name, so `ls` would also work from inside `d`.

## Part 3. Who expands `*`? A program that prints its argv

I wrote `user/args.c` for xv6 (added `$U/_args\` to `UPROGS`) and `args_linux.c` for Linux (compiled with `gcc -o args args_linux.c`). Both print `argc` and every `argv[i]`. I ran the same four lines in both shells.

<p align="center"><img src="screenshots/fig3_1.png" width="600"><br><em>Figure 3.1 – args.c and the UPROGS line</em></p>

<p align="center"><img src="screenshots/fig3_2.png" width="600"><br><em>Figure 3.2 – args output in xv6</em></p>

<p align="center"><img src="screenshots/fig3_3.png" width="600"><br><em>Figure 3.3 – args_linux output in bash</em></p>

| Line | xv6 `argc` and arguments | bash `argc` and arguments |
|---|---|---|
| `args one two three` | 4: `args`, `one`, `two`, `three` | 4: `./args`, `one`, `two`, `three` |
| `args *.c "x y"` | 4: `args`, `*.c`, `"x`, `y"` | 5: `./args`, `a.c`, `args_linux.c`, `b.c`, `x y` |
| `args $HOME` | 2: `args`, `$HOME` | 2: `./args`, `/home/vboxuser` |
| `args 'single quoted'` | 3: `args`, `'single`, `quoted'` | 2: `./args`, `single quoted` |

### Observe

**1. Paste both outputs for `args *.c "x y"`. How many arguments did each program receive?**

xv6:
```
argc = 4
argv[0] = [args]
argv[1] = [*.c]
argv[2] = ["x]
argv[3] = [y"]
```
bash:
```
argc = 5
argv[0] = [./args]
argv[1] = [a.c]
argv[2] = [args_linux.c]
argv[3] = [b.c]
argv[4] = [x y]
```
The xv6 program received 4 arguments (counting its own name, `argv[0]`), and the Linux program received 5.

**2. So who turns `*.c` into file names and removes the quotes: the program, the kernel, or the shell?**

The shell. It is the same program `args` in both cases, and it only prints what it gets. In bash, the shell expanded `*.c` into three file names and removed the quotes around `x y` (keeping it as one argument) before starting the program. The xv6 shell does neither: it passed `*.c` as plain text and split at the spaces, so the quotes stayed in the arguments. The kernel only copies the list of arguments to the new program, it doesn't change it.

**3. What happened to the extra spaces in `one two three`?**

Both shells split the line into words at whitespace and throw the spaces away. With any number of spaces between the words, the program still receives exactly `one`, `two`, `three` as separate arguments, and the spaces themselves are not part of any argument.

## Part 4. Read the shell

```bash
cd ~/xv6-riscv
wc -l user/sh.c
grep -n "#define" user/sh.c
grep -n "case " user/sh.c
grep -n "cd" user/sh.c | head
```

`user/sh.c` has 499 lines, so the whole xv6 shell is about 500 lines long.

<p align="center"><img src="screenshots/fig4_1.png" width="600"><br><em>Figure 4.1 – The command kinds defined in sh.c</em></p>

<p align="center"><img src="screenshots/fig4_2.png" width="600"><br><em>Figure 4.2 – Where each kind is run</em></p>

<p align="center"><img src="screenshots/fig4_3.png" width="600"><br><em>Figure 4.3 – The code that handles cd</em></p>

### Observe

**1. Name the five kinds of command the xv6 shell knows and the character that produces each.**

`sh.c` defines five kinds with `#define` (lines 8 to 12):

| Kind | Produced by | Example |
|---|---|---|
| `EXEC` | a plain command, just words | `echo hello` |
| `REDIR` | `<`, `>` (and `>>`) | `echo hi > f` |
| `PIPE` | `\|` | `ls \| grep READ` |
| `LIST` | `;` | `echo one ; echo two` |
| `BACK` | `&` | `echo bg &` |

Parentheses `( )` only group commands, they don't make a sixth kind. The `case` lines 75 to 125 are in the function that runs each kind (one `case` per kind), and lines 284 to 294 list the special characters the parser recognises: `| ( ) ; & < >`.

**2. Where is `cd` handled, and why is it not run as a separate program?**

`cd` is handled by the shell itself, in the main loop that reads each line, *before* the shell forks a child process. The code checks whether the line starts with `cd ` and then calls `chdir()` (the error message `cannot cd %s` is on line 170). (Replace this with the exact lines from your `sed` output.)

It can't be a separate program because every process has its own current directory. A separate `cd` program would run as a child process: it would change the child's directory, then exit, and the shell would stay in the old directory. Only the shell itself can change its own directory. This is the same reason as in Lab 3, where `cd` is a bash built-in.

The check only looks at the start of the line, which also explains my Part 2 result: in `mkdir d ; cd d ; ls ..` the `cd` came after a `;`, so the shell tried to run a program called `cd` and printed `exec cd failed`.

**3. Search `sh.c` for any code that handles `*` or `$`. What does that explain about Part 2?**

There is none. The search for `$` finds only line 137, `write(2, "$ ", 2)`, which prints the prompt. The `*` search only finds C pointer declarations like `char *argv[MAXARGS]`, and searching for the character `'*'` finds nothing. The list of special characters (lines 284 to 294) has no quotes, `*` or `$` either.

This explains the Part 2 table and Part 3: the xv6 shell can't expand wildcards (`ls *` got a literal `*`), can't expand variables (`echo $HOME` printed `$HOME`), and doesn't treat quotes as special (`"a b"` kept its quotes, and `"x y"` was split into two arguments). All of that is extra work that bash does, and the xv6 shell leaves out.

## Conclusions

In this lab I used the xv6 shell side by side with bash and saw that xv6's shell only does the minimum: it runs programs and supports redirection, pipes, `;`, `&` and grouping. bash adds wildcards, variables, quote handling, history and completion on top. With my `args` program I proved that these things are done by the shell, not by the program or the kernel: the same program received different arguments in each shell. Reading `user/sh.c` showed that the xv6 shell is only about 500 lines, has five kinds of command, and handles `cd` itself because a separate program couldn't change the shell's directory. I also checked that my `sleep` program from Lab 2 already used the new `pause()` name and builds on the current xv6.
