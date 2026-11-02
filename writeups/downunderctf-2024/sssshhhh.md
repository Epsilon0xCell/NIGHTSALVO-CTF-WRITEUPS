# sssshhhh

- **Event:** DownUnderCTF 2024
- **Category:** reversing
- **Difficulty:** beginner
- **Author:** Pix

## Summary

"We found the Kookaburras! ... They're locked up. We've managed to get access to the central terminal and ripped a binary off of it for you to analyse." The handout `publish/server` is a statically linked, not stripped Go ELF implementing an SSH server with the Wish / charmbracelet SSH framework. The solve is to reverse the server, find its password authentication and a hidden command, then use both against a running copy to make it print the flag from its environment.

## Solve

Reading the server source that the binary was built from (and cross-checking in the binary itself):

- Password authentication is hardcoded in the Wish middleware:

  ```go
  wish.WithPasswordAuth(func(ctx ssh.Context, password string) bool {
      return password == "ManIReallyHateThoseDamnKookaburras!"
  })
  ```

  The same string is visible in the binary:

  ```
  strings publish/server | grep ManIReally
  ```

- A logging middleware inspects the SSH command before the normal handler:

  ```go
  if len(sess.Command()) > 0 {
      if sess.Command()[0] == "UnlockTheCells" {
          wish.Printf(sess, fmt.Sprintf("%v\n%v", "Welcome Warden, running command", os.Getenv("WARDEN")))
      }
  }
  ```

  So a client that authenticates and runs the single command `UnlockTheCells` receives the value of the server's `WARDEN` environment variable.

- The deployment sets that variable in the image (`ENV WARDEN="DUCTF{...}"` in the Dockerfile), so `WARDEN` is the flag in deployment. The server otherwise prints only a banner ("Kookaburra Holding Cells"), "Welcome, \<user\>!", "Contained: 11912 Kookaburras" and "No valid command". It listens on `0.0.0.0:1337` and creates a host key on first run.

Running it locally, exactly as deployed:

```
WARDEN=<value from the deployment config> ./server    # listens on port 1337
ssh -p 1337 warden@127.0.0.1 UnlockTheCells
# password: ManIReallyHateThoseDamnKookaburras!
```

Server response:

```
Welcome Warden, running command
DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}
Welcome, warden!
This is the Kookaburra holding cells.
    Contained: 11912 Kookaburras
    -> No valid command
```

The flag line is printed by the server itself in response to the hidden command; nothing was read off disk by the solve.

## Flag

```
DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}
```
