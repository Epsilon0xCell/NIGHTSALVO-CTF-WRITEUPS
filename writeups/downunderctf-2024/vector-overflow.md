# vector overflow

- **Event:** DownUnderCTF 2024
- **Category:** pwn
- **Difficulty:** beginner
- **Author:** joseph

## Summary

"Please overflow into the vector and control it!" A small NoPIE C++ binary reads input into a fixed buffer with an unbounded `std::cin >> buf`, right next to a `std::vector` whose contents decide whether you reach the win path.

## Solve

Layout in the binary's data section:

- the input buffer `buf` sits at `0x4051e0`
- the `std::vector` object `v` sits at `0x4051f0`, immediately after it: begin pointer, end pointer, capacity pointer

The read has no length limit, so input longer than the buffer overwrites the vector's three pointers. The program later reads the vector's elements and compares them against the expected string `DUCTF`. The trick is to aim the vector back into the buffer itself:

1. Fill the start of `buf` with the bytes `DUCTF` (the data the vector will "contain").
2. Keep writing past the buffer so the vector's begin pointer is overwritten with the address of `buf` (`0x4051e0`).
3. The element check now reads `DUCTF` out of our own input, the check passes, and the program drops into its win path and a shell.

One practical wrinkle: `std::cin` buffers stdin, so the exploit has to be interactive. Send the overflow first, wait for the shell to spawn, and only then send the command; piping everything in one write gets the command swallowed by the input stream before the shell exists.

```
$ ./vector-overflow        # with the crafted overflow, then after a beat:
$ cat flag.txt
```

## Flag

```
DUCTF{y0u_pwn3d_th4t_vect0r!!}
```
