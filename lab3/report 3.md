# LAB 3 – Report

Zhuchkova Sofia FAF-241

## Purpose of the work

To learn to work fast in the bash shell on Linux, to understand what the shell does to a command line before it runs (globbing, quoting, expansion), to redirect output and chain commands, to answer questions about data with pipes of small filters (grep, cut, sort, uniq, wc), and to write a first shell script.

## Part 1

1. Which of cd, ls, echo are built-ins, and why MUST cd be one?
`cd` and `echo` are shell built-ins (`type` says "is a shell builtin"). `ls` is an external program, a separate file on disk (`type` shows its path, or an alias that points to it).
`cd` must be a built-in because every process has its own current directory. An external program runs as a child process, so if `cd` were external it would change the directory of the child, which then exits, and the shell would stay where it was. Only the shell itself can change its own directory.

2. How many directories are in your PATH, and in which one is ls?
My PATH has 10 directories (counted with `echo $PATH | tr ':' '\n' | wc -l`). The `ls` command is in `/usr/bin/ls` (shown by `which ls`).

3. What exit code does a command that is not found return?
It returns 127 ("command not found"). Successful commands return 0, and other errors return other non-zero numbers.

4. Which option of ls sorts by modification time?
`ls -t` sorts by modification time, newest first. `ls -lt` does the same with details, and `ls -ltr` reverses the order (oldest first).

## Part 2
1. 
| Command | My prediction | Real output | Explanation |
|---|---|---|---|
| `echo *.c` | `.c` (not sure?) | `a.c b.c` | The shell replaces `*.c` with all matching file names before echo runs. |
| `echo file?.txt` | `file?.txt` | `file1.txt file2.txt` | `?` matches exactly one character. |
| `echo [a-b]*` | `[a-b]` (again not sure) | `a.c alfa beta.txt b.c` | Matches names starting with a or b, sorted alphabetically. |
| `echo *.xyz` | `*.xyz` | `*.xyz` | Nothing matches, so the shell leaves the pattern as plain text. |
| `ls alfa beta.txt` | error probably, no file like that | `ls: cannot access 'alfa': No such file or directory` and the same for `'beta.txt'` | The shell splits the line at the space, so ls got two names. |
| `ls "alfa beta.txt"` | `alfa beta.txt` (?) | `'alfa beta.txt'` | The quotes keep the name as one word. |
| `echo '$HOME' "$HOME" \$HOME` | no idea | `$HOME /home/vboxuser $HOME` | Single quotes and backslash print it literally. Double quotes expand the variable. |
| `echo {1..5} ; echo $((7 * 6))` | nums 1-5 and 42 | `1 2 3 4 5` and `42` | `{1..5}` expands to numbers, `$(( ))` calculates. |
| `echo "today is $(date +%A), I am $(whoami)"` | today's date and vbox user's name | `today is Sunday, I am vboxuser` | `$( )` runs the command and inserts its output. |
| `touch log{1..3}.txt && ls log*` | probably the logs 1-3? | `log1.txt log2.txt log3.txt` | `{1..3}` makes three names. `&&` runs `ls` only if `touch` succeeded. |

2. echo knows nothing about files, yet `echo *.c` printed file names. Who produced them?
The shell did. Before running echo, the shell expands `*.c` (globbing) into the list of matching files, so echo just receives `a.c b.c` as ordinary words and prints them.

3. Why did `ls alfa beta.txt` fail? Give two ways to fix it.
The shell splits a command line into words at spaces, so ls received two arguments, `alfa` and `beta.txt`, and neither file exists. Two fixes:
- Quotes: `ls "alfa beta.txt"`
- Backslash before the space: `ls alfa\ beta.txt`

4. What is the difference between single and double quotes?
Single quotes print everything exactly as typed, with no expansion at all. Double quotes also keep spaces together, but the shell still replaces `$VARIABLE` and `$(command)` inside them. That is why `'$HOME'` printed `$HOME` while `"$HOME"` printed `/home/vboxuser`.

Why does ls show 'alfa beta.txt' in quotes?
ls adds quotes around names that contain special characters such as spaces, so you can see it is one name and copy it safely. The quotes are not part of the real file name.

## Part 3
1. Which file got the error message, and which got the listing?

The listing of `/etc` went to `out.txt` (237 lines), because `>` redirects stdout (the normal output). The error message (`ls: cannot access '/nope'...`) went to `err.txt`, because `2>` redirects stderr (the error channel). The two kinds of output are separate streams, so they can be saved separately.

2. What is the difference between `> all.txt 2>&1` and `2>&1 > all.txt`?

The shell reads redirections from left to right, and `2>&1` means "send stderr to wherever stdout points *right now*".
- `ls /etc /nope > all.txt 2>&1`: first stdout goes to the file `all.txt`. Then stderr is sent to the same place as stdout, so it also goes to the file. Result: both the listing and the error are in `all.txt`, and nothing is printed on screen.
- `ls /etc /nope 2>&1 > all.txt`: first stderr is sent to where stdout points now, which is still the terminal. Only after that is stdout moved to the file. Result: the listing is in `all.txt`, but the error message was already connected to the terminal, so it appeared on the screen.
The result differs only because the order of the redirections is different.

3. Why does `wc -l < access.log` print only a number, while `wc -l access.log` also prints the file name?
With `wc -l access.log`, wc receives the file name as an argument, opens the file itself, and so it knows the name and prints it. With `wc -l < access.log`, the shell opens the file and feeds its content to wc as standard input. wc just sees a stream of text with no name, so it prints only the number (500).

4. Explain each operator in one sentence:
- `;` runs the next command after the first one finishes, no matter whether the first one succeeded or failed.
- `&&` runs the next command only if the previous one succeeded (exit code 0).
- `||` runs the next command only if the previous one failed (non-zero exit code).

Extra notes for the report
- `ls /nope 2> /dev/null || echo "failed with code $?"` printed `failed with code 2`. The error message was thrown away into `/dev/null`, and `||` ran the echo because ls failed. Exit code 2 is what `ls` returns for a serious error such as a missing file.
- `sleep 100 &` started a background job (job number `[1]`, process ID 8472). `jobs` showed it as Running, `kill %1` stopped it, and the next `jobs` showed it as Terminated.

## Part 4
# Q1. How many requests failed?
```bash
grep -c ' FAIL ' access.log
```
Answer: **100**. `grep -c` counts the lines that contain the word FAIL.

# Q2. How many different pages were requested, and which?
```bash
cut -d' ' -f5 access.log | sort -u | wc -l ; cut -d' ' -f5 access.log | sort -u
```
Answer: **4 pages**: `/page0`, `/page1`, `/page2`, `/page3`. `cut` takes the 5th column (the page), `sort -u` sorts and removes duplicates, and `wc -l` counts what is left.

# Q3. How many requests did each page get?
```bash
cut -d' ' -f5 access.log | sort | uniq -c
```
Answer: **125 requests per page** (4 × 125 = 500, so the total matches the log). `sort` puts equal lines next to each other, and `uniq -c` counts each group.

# Q4. Which user or users have the most failed requests, and how many?
```bash
grep ' FAIL ' access.log | cut -d' ' -f3 | sort | uniq -c | sort -rn | head -2
```
Answer: **user3 and user5 are tied with 15 failures each.** The other five users have 14 each. The check works: 15 + 15 + 5 × 14 = 100, which equals Q1. `sort -rn` sorts the counts from biggest to smallest.

# Q5. How many requests did user3 make, and how many of them failed?
```bash
grep -c ' user3 ' access.log ; grep ' user3 ' access.log | grep -c ' FAIL '
```
Answer: user3 made **72** requests, and **15** of them failed. The second command filters twice with a pipe: first the user3 lines, then only the FAIL ones among them.

# Q6. Print the last 3 failed requests, showing only time and user.
```bash
grep ' FAIL ' access.log | tail -3 | cut -d' ' -f2,3
```
Answer:
```
10:10 user0
10:15 user5
10:20 user3
```
`tail -3` keeps the last 3 lines, and `cut -f2,3` keeps only the time and user columns.

# Q7. Which login shells appear in /etc/passwd, and how many accounts use each?

```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```
Answer:
```
     40 /usr/sbin/nologin
      5 /bin/false
      2 /bin/bash
      1 /bin/sync
```
Field 7 of `/etc/passwd` is the login shell, and the separator is `:`, so `cut -d:` is used. Most accounts are system accounts that can't log in (`nologin`, `false`).

# Q8. Why do `ls /etc | wc -l` and `ls -l /etc | wc -l` differ by exactly one?
my prediction: the `-l` command probably adds something like an additional line or reads the name of the file?

Output: `236` and `237`.
The long format `ls -l` prints one extra first line, `total N`, which shows the disk space used by the listed files. It is not a file, but `wc -l` counts it as a line, so the number is higher by exactly one.

# Bonus
Lines where the minute ends in 0 and the request failed:
```bash
grep -E '10:[0-5]0 user[0-9] FAIL' access.log | wc -l
```
Answer: **50**. `[0-5]0` matches the minutes 00, 10, 20, 30, 40 and 50. All 50 lines whose minute ends in 0 are also FAIL lines, because those request numbers are multiples of 10, and so also of 5.

## Part 5
The completed loop in `report.sh`:

```bash
for n in 0 1 2 3 4 5 6; do
  c=$(grep "user$n" "$log" | grep -c FAIL)
  echo "user$n: $c failures"
done
```

For each user number, `grep` takes that user's lines, and `grep -c FAIL` counts how many of them failed.
Output: `500 requests, 100 failed`, then user3 and user5 with 15 failures each and the other users with 14 each. For a missing file: `no such file: nope.log` and `exit code: 1`.

1. What does the first line `#!/bin/bash` do, and what happens if you run `./report.sh` before `chmod +x`?
The first line (called a shebang) tells the system which program should interpret the file, here bash.
Before `chmod +x` the file does not have the executable permission, so `./report.sh` fails with "Permission denied". `chmod +x` adds that permission.

2. Why must you type `./report.sh` and not just `report.sh`?
When you type a bare command name, the shell looks for it only in the directories listed in `PATH`. The current directory is not in `PATH` (for security, so a random file in a folder can't replace a real command). `./` gives an explicit path ("the file here in the current directory"), so the shell runs it directly without searching.

3. Why does the script write the error with `>&2` and end with `exit 1`?
`>&2` sends the message to stderr (the error channel) instead of normal output. This way, errors don't get mixed with the real results if the output is redirected to a file or piped into another command, and they still show on the screen.
`exit 1` stops the script and returns exit code 1, which means "failure" (0 means success). Other programs and scripts can check it with `$?` or use `&&` / `||`. Without it the script would not stop at that point, and would not report failure.

# Bonus
Crontab line (added with `crontab -e`):
```
* * * * * /home/vboxuser/os-lab3/report.sh /home/vboxuser/os-lab3/access.log >> /home/vboxuser/os-lab3/report.log 2>&1
```
The five `*` mean "every minute of every hour, day, month and weekday". Then comes the command with full paths, because cron does not start in my folder and has no idea where `report.sh` or `access.log` are. `>>` appends the output to `report.log` instead of overwriting it, and `2>&1` sends errors to the same file.
Content of `report.log` after waiting a few minutes: the report appeared once per minute (I waited about 5 minutes, so it was written 5 times):

```
500 requests, 100 failed
user0: 14 failures
user1: 14 failures
user2: 14 failures
user3: 15 failures
user4: 14 failures
user5: 15 failures
user6: 14 failures
(...the same block repeated for each run)
```
After the test I removed the line with `crontab -r`, and `crontab -l` now says `no crontab for vboxuser`.

## Conclusions

In this lab I learned that the shell, not the command, expands patterns like `*.c` and variables like `$HOME`, which is why quotes matter. I saw that stdout and stderr are separate streams and that the order of redirections changes the result (`> file 2>&1` vs `2>&1 > file`). With one-line pipes I answered all the questions about the access log, and the totals checked out (500 requests, 100 failures). Finally I turned the one-liners into a script with a loop, an error check and an exit code, and ran it automatically with cron.
