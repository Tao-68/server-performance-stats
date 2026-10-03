# server-performance-stats

This is part of [roadmaps.sh](https://roadmap.sh/projects/server-stats) DevOps project.

Tools used throughout:

| Tool | What it does |
|---|---|
| `grep` | keeps only the lines that match a word |
| `cut` | cuts a line into pieces and keeps some of them |
| `awk` | splits each line into columns (`$1`, `$2`, ...) and does maths or printing |
| `head -n 6` | keeps only the first 6 lines |

Most commands below use a pipe `|`. A pipe sends the output of the command on the left into the command on the right.

---

## 1. Total CPU usage

```bash
top -bn2 | grep "%Cpu(s):" | tail -n 1 | cut -d ',' -f 4 | awk '{print "Usage: " 100-$1 "%"}'
```

- `top` is a live monitor. Normally it keeps refreshing, so two flags make it print and exit:
- `-b` batch mode: plain text output that can be piped.
- `-n2` take 2 snapshots, then stop.

Why 2 snapshots? The first snapshot is an average since the machine booted, so it is not very useful. The second snapshot covers the last few seconds, and that is the number we really want.

`top` prints many lines. Example:
```bash
top - 12:21:37 up  8:22,  2 users,  load average: 0.12, 0.12, 0.04
Tasks:  67 total,   2 running,  61 sleeping,   4 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
MiB Mem :   7902.6 total,   4692.3 free,   2459.8 used,   1004.3 buff/cache     
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   5442.8 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
```

`grep "%Cpu(s):"` keeps only the CPU summary lines, which look like:

```
%Cpu(s):  0.4 us,  0.4 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
```
Each pair is a percentage of CPU time followed by a label for what the CPU was doing during that period.

- `us` (user): Runing normal programs/ applications (non-kernel code)
- `sy` (system): Running the Linux kernel's work for programs (file access, networking etc.)
- `ni` (nice): Running user programs that were given a lower priority with the nice command. Usually 0.
- `id` (idle): Doing nothing.
- `wa` (I/O wait): Waiting for the disk or network to respond. High values mean slow storage.
- `hi` (hardware interrupts): Handling signals from hardware (disk, network card, keyboard).
- `si` (software interrupts): Handling signals raised by the kernel itself, such as network packet processing.
- `st` (steal): Time the virtual machine wanted the CPU but the host took it for something else. Only matters on VMs and cloud servers (and WSL), and is usually 0.

Using `cut -d ',' -f 4`**

- `-d ','` split the line at every comma. 

(Note: might cause error if lokale uses `,` for decimal place)
- `-f 4` keep only the 4th piece.

The pieces are:

| Piece | Text |
|---|---|
| 1 | `%Cpu(s):  0.4 us` |
| 2 | ` 0.4 sy` |
| 3 | ` 0.0 ni` |
| 4 | ` 99.0 id` (this is the idle one) |

The result is ` 99.0 id`.

`awk '{print "Usage: " 100-$1 "%"}'`**

- `$1` is the first word of the line, which is `99.0`.
- `100-$1` is `100 - 99.0 = 1`.
- The text around it prints `Usage: 1%`.

Because `top -bn2` takes 2 snapshots, the script prints **two** `Usage:` lines. The first is the average since boot and the second is the current usage. To print only the current one, add `tail -n 1` before the awk:

`tail -n 1` keeps only the last line (the second snapshot).

---

## 2. Total memory usage

```bash
free | grep "Mem:" -w | awk '{printf "Total: %.1fGi\nUsed: %.1fGi (%.2f%%)\nFree: %.1fGi (%.2f%%)\n",$2/1024^2, $3/1024^2, $3/$2 * 100, $4/1024^2, $4/$2 * 100}'
```

- `free` shows memory in kilobytes. Its `Mem:` line has the columns: `$2` total, `$3` used, `$4` free.
- `grep "Mem:" -w` keeps only the `Mem:` line (not the swap line).
- `/1024^2` is the same as dividing by 1024 twice: KB to MB to GB.
- Percentages are `used / total * 100` and `free / total * 100`.

---

## 3. Total disk usage

```bash
df -k / | awk 'NR==2 {t=$3+$4; printf "Total: %.1fG\nUsed: %.1fG (%.2f%%)\nFree: %.1fG (%.2f%%)\n", t/1024/1024, $3/1024/1024, $3/t*100, $4/1024/1024, $4/t*100}'
```

- `df` means "disk free". `-k` prints sizes in kilobytes (exact numbers, easier for maths). `/` limits it to the main disk only.
- `NR==2` means "only line 2", which skips the header line.
- Columns: `$2` Size, `$3` Used, `$4` Available.
- `t=$3+$4` stores Used + Available in a variable `t` and uses it as the total.
- `/1024/1024` converts KB to GB.

### Why Total = Used + Available, not the Size column?

On Linux, about 5% of the disk is **reserved** for the system. That space is counted in `Size` but not in `Used` or `Available`. Using Used + Available makes the percentages add up to 100%.

### Why not `df -h --total`?

`--total` adds up every mounted filesystem. In WSL that includes the Windows drives (`/mnt/c`) and virtual mounts, so the number is much bigger than the real Linux disk.

### How `printf` fills its slots

```
printf "FORMAT" , value1, value2, value3 ...
```

Each `%s` or `%.1f` in the format is a slot. Values fill the slots **in order**, left to right. `%.1f` means a decimal with 1 digit after the point, and `%%` prints a literal `%` (it is not a slot).

---

## 4. Top 5 processes by memory

```bash
ps aux --sort -%mem | head -n 6 | awk '{print $1 "\t" $2 "\t" $4 "\t" $11}'
```

- `ps aux` lists every running process. `a` all users, `u` show usage columns, `x` include background processes.
- `--sort -%mem` sorts by memory, highest first (the `-` means descending).
- `head -n 6` keeps the header plus the top 5 processes.
- awk columns: `$1` user, `$2` PID, `$4` memory %, `$11` command.

### Why `$11` shows only part of the command

awk splits on spaces, and the command can contain many words. `$11` is only the first word (the program path). The arguments are in `$12`, `$13`, and so on.

To show the full value in field `COMMAND`:

```bash
ps aux --sort=-%mem | head -n 6 | awk '{printf "%s\t%s\t%s\t", $1, $2, $4; for(i=11;i<=NF;i++) printf "%s ", $i; print ""}'
```

`NF` is the number of fields on the line, so the loop prints field 11 to the end.

### Cleaner alternative

```bash
ps -eo user,pid,%mem,args --sort=-%mem | head -n 6
```

`-o` lets you pick the columns directly. Use `args` for the full command, or `comm` for the short name. Note that `comm` can show unhelpful names like `MainThread`.

---

## 5. Top 5 processes by CPU

```bash
ps aux --sort -%cpu | head -n 6 | awk '{print $1 "\t" $2 "\t" $3 "\t" $11}'
```

Same as the memory version, but sorted by `%cpu`, and it prints `$3` (CPU %) instead of `$4` (memory %).

---

## Quick reference

| Goal | Command |
|---|---|
| Live process monitor | `top` (press `q` to quit, `P` sort by CPU, `M` sort by memory) |
| Disk usage, readable | `df -h /` |
| Disk usage in GB (rounded) | `df -BG /` |
| Memory usage | `free -h` |
| Top processes by memory | `ps aux --sort=-%mem \| head -n 6` |
