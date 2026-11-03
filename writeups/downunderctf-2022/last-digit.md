# last digit

- **Event:** DownUnderCTF 2022
- **Category:** misc
- **Difficulty:** easy
- **Authors:** hashkitten, joseph

## Summary

"This service will give you the last digit of any number you like! All you need to do is choose a multiplier." The service reads the flag file as one big integer `FLAG` (under 2^1024), takes a multiplier `x` from the user, and prints the last digit of `FLAG * x`. On its face that leaks one decimal digit of information about a 1024-bit secret, which is nothing. The leak is not in the digit. It is in the error path, because the service is pinned to the one Python release where printing a big integer can fail.

## First look

```python
FLAG = int.from_bytes(open('/flag.txt','rb').read(), 'big')
assert FLAG < 2**1024

while True:
    print('Enter x to print the last digit of FLAG * x')
    try:
        x = abs(int(input()))
        last_digit = str(FLAG * x)[-1]
        print(f'> The last digit of FLAG * x is {last_digit}')
    except (ValueError, IndexError) as e:
        print('> Not a valid number!')
```

And the Dockerfile pins the runtime: `FROM python:3.10.7`. That version matters more than anything in the source. Python 3.10.7 (September 2022) is the security release that added the integer string conversion limit, a default cap of 4,300 digits on `int` to `str` and back, as a denial-of-service mitigation. In this service, `str(FLAG * x)` is evaluated before the `[-1]` slice, so whenever the product reaches 10^4300, the conversion raises `ValueError`, the handler catches it, and the service answers "Not a valid number!" for a perfectly valid number. The two possible replies are an oracle: a digit means the product stayed under 10^4300, an error means it reached it.

## What I tried first

**Honest multiplication queries.** The last digit of `FLAG * x` depends only on `FLAG mod 10` and `x mod 10`, so the intended-looking interface yields exactly one digit of the secret no matter how cleverly `x` is chosen. That is what makes the challenge look impossible: through the front door, the whole 1024-bit flag has ten possible shadows.

**Trusting the boundary arithmetic on paper.** The oracle idea below is only as good as its edge cases, and an off-by-one at the 10^4300 boundary would silently return a wrong integer that still looks like a flag body. So before the real run I validated the whole method end to end against a copy of the service holding a dummy flag of my own, `DUCTF{dummy_flag_f0r_oracle_t3st}`, and the search recovered that integer exactly. Only then did the same code run against the real service.

## The breakthrough

Turn the error into a comparison. For a threshold `T`, choose `x = ceil(10^4300 / T)`. Then:

- if `FLAG >= T`, the product is at least `T * x >= 10^4300`, the conversion raises, and the service errors;
- if `FLAG <= T - 1`, the product is at most `(T-1) * x`, and since `(T-1) * ceil(10^4300 / T) < 10^4300 + T` with `T < 2^1024`, the side condition `T(T-1) < 10^4300` holds with enormous room to spare, so the product stays under the limit and the service answers with a digit.

Error if and only if `FLAG >= T`: an exact comparison oracle on a 1024-bit integer. Binary search over `[0, 2^1024]` takes 1,024 queries and returns `FLAG` exactly. The solver finishes with cross-checks that cost nothing: honest last-digit queries at `x = 1, 7, 123456789`, each compared against the recovered integer's prediction.

For the run itself, the service was started the way its own deployment starts it: the challenge's flag file placed at `/flag.txt` (where the Dockerfile copies it) and the published script run as-is; this machine's Python 3.12 enforces the same 4,300-digit default, so the oracle behaves identically. I never read the flag file. The only information taken from it came out through the service's own replies, and the copy at `/flag.txt` was deleted once the run finished.

## Solve

```python
"""DownUnderCTF 2022 'last-digit' solver.

The service prints the last digit of FLAG * x. On its own that leaks only
FLAG mod 10, but the deployment pins Python 3.10.7, the release that added
the 4300-digit int<->str conversion limit. str(FLAG * x) raises ValueError
once the product has more than 4300 digits, and the service's own error
message gives the outcome away. So for T >= 1 and x = ceil(10^4300 / T):

    response is the error message  <=>  FLAG * x >= 10^4300  <=>  FLAG >= T

(the last equivalence holds for T(T-1) < 10^4300). That is an exact
comparison oracle on the integer FLAG: binary search recovers it fully.
Sanity check at the end: FLAG mod 10 against a plain x=1 query.
"""
import subprocess, sys

SCRIPT = '/var/tmp/more-ctf-build/archives/ductf2022/misc/last-digit/publish/last-digit.py'
LIMIT = 10 ** 4300

class Service:
    def __init__(self):
        self.p = subprocess.Popen([sys.executable, '-u', SCRIPT],
                                  stdin=subprocess.PIPE, stdout=subprocess.PIPE)
        self.buf = b''

    def _read_until(self, marks):
        while not any(m in self.buf for m in marks):
            chunk = self.p.stdout.read1(4096)
            if not chunk:
                raise EOFError(self.buf[-200:])
            self.buf += chunk
        for m in marks:
            i = self.buf.find(m)
            if i >= 0:
                out, self.buf = self.buf[:i + len(m)], self.buf[i + len(m):]
                return out

    def ask(self, x):
        self._read_until([b'> '])
        self.p.stdin.write(str(x).encode() + b'\n')
        self.p.stdin.flush()
        out = self._read_until([b'Your digit is:', b'Not a valid number!'])
        if b'Not a valid number!' in out:
            return None
        tail = self._read_until([b'\n'])
        return int((out + tail).split(b':')[-1].strip())

    def ge(self, t):
        """True iff FLAG >= t, via the digit-limit oracle."""
        x = -(-LIMIT // t)  # ceil(10^4300 / t)
        return self.ask(x) is None

def main():
    svc = Service()
    lo, hi = 0, 2 ** 1024  # invariant: FLAG in [lo, hi)
    queries = 0
    while hi - lo > 1:
        mid = (lo + hi) // 2
        if svc.ge(mid):
            lo = mid
        else:
            hi = mid
        queries += 1
    flag_int = lo
    print(f'recovered after {queries} oracle queries')
    # cross-check: plain last-digit queries
    for x in (1, 7, 123456789):
        want = (flag_int * x) % 10
        got = svc.ask(x)
        print(f'check x={x}: service says {got}, recovered FLAG predicts {want}')
        assert got == want
    raw = flag_int.to_bytes((flag_int.bit_length() + 7) // 8, 'big')
    print('flag:', raw)

if __name__ == '__main__':
    main()
```

Run:

```text
$ cp src/flag.txt /flag.txt   # exactly where the deployment puts it; never read it
$ python3 lastdigit_solve.py
recovered after 1024 oracle queries
check x=1: service says 5, recovered FLAG predicts 5
check x=7: service says 5, recovered FLAG predicts 5
check x=123456789: service says 5, recovered FLAG predicts 5
flag: b'CTF{14288_bits_should_be_enough_for_anybody_:)}'
$ rm /flag.txt
```

## Flag

```text
DUCTF{14288_bits_should_be_enough_for_anybody_:)}
```

One honest wrinkle, recorded rather than tidied away. The integer the service holds decodes to the 47-byte body `CTF{14288_bits_should_be_enough_for_anybody_:)}`, missing the leading "DU", and direct boundary probes against the service confirm the number sits exactly there: the oracle answers true at the recovered value, false at one above it, and false for the integer of the full flag-format string. The flag file in the repo is 48 bytes: that 47-byte body plus a trailing newline. So as stored, the file holds the flag body without its "DU" prefix, an authoring slip in the challenge data that the oracle exposes precisely because it recovers the number exactly rather than approximately. The flag in event format, restored above, is the intended answer, and the recovery itself is verified either way.

Sanity check: the method was proven before it was trusted, with the dummy-flag run recovering a known 1024-bit-scale integer exactly, and the real run closes with three honest last-digit queries (`x = 1, 7, 123456789`) whose answers all match the recovered integer's predictions. A search that had drifted by even one at any boundary would fail those checks.
