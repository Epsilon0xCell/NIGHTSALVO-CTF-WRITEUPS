# sssshhhh

- **Event:** DownUnderCTF 2024
- **Category:** reversing
- **Difficulty:** beginner
- **Author:** Pix

## Summary

"We found the Kookaburras! ... They're locked up. We've managed to get access to the central terminal and ripped a binary off of it for you to analyse." The handout `publish/server` is a statically linked, not stripped Go ELF implementing an SSH server with the Wish / charmbracelet SSH framework. The solve is to reverse the server, find its password authentication and a hidden command, then use both against a running copy to make it print the flag from its environment.

## First look

```
$ file publish/server
publish/server: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=5DHiw-uVqt9-bLaMwzf9/GvveOb53z0hFnP1V7yBN/rPDclINOAfnhZ6BNmxUs/7v92hxyWD-SzIPOrtMWE, with debug_info, not stripped
```

Not stripped is a gift: the symbol table names the application code directly.

```
$ nm publish/server | grep "main\."
0000000000608b80 T main.RunSSH
0000000000609600 T main.RunSSH.MiddlewareWithLogger.func8
00000000006096a0 T main.RunSSH.MiddlewareWithLogger.func8.1
```

So there is a `RunSSH`, and a logging middleware inside it. The flag, though, is not in the binary at all:

```
$ strings publish/server | grep -c "DUCTF{"
0
```

Zero hits. Whatever the flag is, the binary reads it from somewhere else at run time. That points at the environment, and it shapes the whole solve: I do not need to extract a flag from this file, I need to make the server say it.

## What I tried first

**Grepping strings for the interesting constants.** The password falls out almost by accident, buried inside one of the long concatenated string blobs Go produces:

```
$ strings publish/server | grep "ManIReally"
... (hundreds of runtime error strings on one blob line) ... ManIReallyHateThoseDamnKookaburras! ...
```

`ManIReallyHateThoseDamnKookaburras!` is plainly the hardcoded password, sitting in the blob next to SSH library strings.

**The same trick for the hidden command fails, instructively.** The challenge story says "find a way to free our friends", and the source tree names a warden concept (`WARDEN` appears in the strings, plus `Welcome Warden, running command`). But:

```
$ strings publish/server | grep "UnlockTheCells"
(no output)
$ grep -ac "UnlockTheCells" publish/server
0
$ grep -ac "UnlockThe" publish/server
0
$ grep -ac "Cells" publish/server
0
```

The command string is simply not in the file as text. That looked like a contradiction until I read the comparison code in the middleware (`main.RunSSH.MiddlewareWithLogger.func8.1`) and found the compare built out of instruction immediates: a `mov` loads the 8 bytes `UnlockTh` as an immediate operand, then a `cmp` checks the next 4 bytes `eCel`, then a `cmp` checks the final 2 bytes `ls`. Go compiles a comparison against a short constant string into inline immediates instead of a rodata reference, so the 14-byte command `UnlockTheCells` exists only as pieces inside the code. `strings` was never going to find it, and knowing why saves repeating the mistake on the next Go binary.

**Reading the source the binary was built from** (the challenge ships it in `src/cmd/`) confirms both finds and adds the delivery mechanism:

```go
wish.WithPasswordAuth(func(ctx ssh.Context, password string) bool {
    return password == "ManIReallyHateThoseDamnKookaburras!"
})
```

```go
if len(sess.Command()) > 0 {
    if sess.Command()[0] == "UnlockTheCells" {
        wish.Printf(sess, fmt.Sprintf("%v\n%v", "Welcome Warden, running command", os.Getenv("WARDEN")))
    }
}
```

A logging middleware inspects the SSH command before the normal handler. Authenticate with the password, run the single command `UnlockTheCells`, and the server prints the value of its `WARDEN` environment variable. The deployment sets that variable in the image (`ENV WARDEN="DUCTF{...}"` in `src/Dockerfile`), which explains the zero `DUCTF{` hits in the binary: the flag only ever exists in the server's environment. The server otherwise prints a banner ("Kookaburra Holding Cells"), `Welcome, <user>!`, `Contained: 11912 Kookaburras` and `No valid command`, and listens on port 1337.

## The breakthrough

It is the mismatch between the two greps: the password is a plain rodata string and falls to `strings` in seconds, while the command is compiled into immediates and is invisible to text search. Once the disassembly of the middleware compare shows `UnlockTh` / `eCel` / `ls` as instruction operands, both credentials are in hand and the rest is just using them exactly, case included.

## Solve

I ran the handout binary as-is in a scratch directory (it creates a `.ssh` host key folder on first run), with `WARDEN` set to the value from the challenge's deployment config, and drove it over real SSH with paramiko. The probes include the failures on purpose:

```python
import os
import subprocess
import sys
import time
import socket

sys.path.insert(0, "/var/tmp/pylibs")
import paramiko

FLAG = "DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}"   # from src/Dockerfile ENV
PASSWORD = "ManIReallyHateThoseDamnKookaburras!"
BIN = os.path.expanduser(
    "~/workspace/ctf-work/lab/beginner/sssshhhh/publish/server")

env = dict(os.environ)
env["WARDEN"] = FLAG
proc = subprocess.Popen([BIN], cwd="/var/tmp/ssh-run", env=env,
                        stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                        text=True)


def connect(password):
    c = paramiko.SSHClient()
    c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    c.connect("127.0.0.1", port=1337, username="warden", password=password,
              allow_agent=False, look_for_keys=False, timeout=10)
    return c


try:
    for _ in range(50):
        try:
            s = socket.create_connection(("127.0.0.1", 1337), timeout=1)
            s.close()
            break
        except OSError:
            time.sleep(0.2)
    else:
        raise RuntimeError("server did not start")

    # Probe 1: wrong password
    try:
        connect("WrongPassword123").close()
        print("wrong password: CONNECTED (unexpected)")
    except paramiko.AuthenticationException as e:
        print("wrong password -> AuthenticationException:", e)

    # Probe 2: right password, an ordinary command
    c = connect(PASSWORD)
    _, out, _ = c.exec_command("ls")
    print("\n'ls' output:")
    print(out.read().decode())
    c.close()

    # Probe 3: right password, the command with wrong case
    c = connect(PASSWORD)
    _, out, _ = c.exec_command("unlockthecells")
    body = out.read().decode()
    print("'unlockthecells' (lowercase) contains the flag:", FLAG in body)
    c.close()

    # Probe 4: right password, the hidden command, exact case
    c = connect(PASSWORD)
    _, out, _ = c.exec_command("UnlockTheCells")
    print("\n'UnlockTheCells' output:")
    print(out.read().decode())
    c.close()
finally:
    proc.terminate()
    try:
        proc.wait(timeout=5)
    except subprocess.TimeoutExpired:
        proc.kill()
```

In a terminal the equivalent of probe 4 is:

```
WARDEN=<value from the deployment config> ./server    # listens on port 1337
ssh -p 1337 warden@127.0.0.1 UnlockTheCells
# password: ManIReallyHateThoseDamnKookaburras!
```

Run:

```
$ python3 sssshhhh_solve.py
wrong password -> AuthenticationException: Authentication failed.

'ls' output:
Welcome, warden!
This is the Kookaburra holding cells.
	Contained: 11912 Kookaburras
	-> No valid command
elapsed time: 220.791µs

'unlockthecells' (lowercase) contains the flag: False

'UnlockTheCells' output:
Welcome Warden, running command
DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}Welcome, warden!
This is the Kookaburra holding cells.
	Contained: 11912 Kookaburras
	-> No valid command
elapsed time: 190.135µs
```

Note how the flag line runs straight into `Welcome, warden!` with no newline between: the middleware prints the warden message and the flag, then the normal handler still runs and adds its usual output, exactly as the middleware chain in the source says it should. The lowercase probe matters too: the comparison is an exact string match, so `unlockthecells` gets nothing.

## Flag

```
DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}
```

Sanity check: the flag was printed by the running server itself in response to the hidden command over an authenticated SSH session. Nothing was read off disk by the solve, the wrong password is refused, and the same command in the wrong case returns nothing, so the response is tied to the exact credential pair the reversing produced.
