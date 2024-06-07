---
layout: post
title:  "Bash built-in sleep"
date:   2024-06-07 16:00:00 +0800
categories: scripting
---

## Background

If you write a lot of Bash scripts and have implemented a simple for-each-interval pattern, you've probably used a lot of loops and `sleep` in the loops.

The pattern usually looks like this:
```sh
preparation_for_jobs
while true; do
    check_for_some_flag_and_do_some_thing
    sleep 5
done
```

In most cases the script would be long-running, and it essentially does what a `systemd.timer` or `cron` job would do naively in Bash. But hey, we do have some context before the loop that those don't provide, and we really want to save `fork()` calls and a lot of PIDs wasted by restarting a script over and over again in those supervisors.

But after running the above script for a long time, you'll notice that your PIDs are being wasted fast! What's the naughty guy that take your precious PIDs one by one and throw it on ground?

```sh
> ps
```
```
systemd─┬─dbus-broker-lau───dbus-broker
         └─systemd─┬─(sd-pam)
                      └─yakuake────fish───bash───sleep
```
_(Many other processes are omitted)_

```sh
> type sleep
```
```
sleep is /usr/bin/sleep
```

Oh it's `sleep`! It's used so often in the script, and as it's not a builtin, every time you `sleep` you waste a PID...

Well `sleep` is not a builtin, but an external program. It's designed like so for many reasons and let's not turn this into a long UNIX history introduction.

What if we want a builtin `sleep`?

## Hacky solution

The following is the solution I found today in `#archlinuxcn` group, which looks very hacky and certainly would be a good PID saver right?

```sh
sleep(){
    read -t "$1" <> <(:)
}
```

Basically the idead behind this is:
- `read` is a Bash built-in to read from stdin and store them into Bash variables
- `read` has an optional `-t` argument that would wait for at most specified time duration
- `(:)` would spawn a subshell, and as `:` is a no-op the subshell does nothing and exits right away
- The `<` in `<(:)` would cause the whole `<(:)` "argument" to be substituted into the corresponding fd path, e.g. `/dev/fd/63`, providing the subshell's stdout to be readable in parent
- The `<>` redirects both stdout and stdin of command `read` into the later path, in this case the subshell
- As the subshell dies right away, and the `read` command wants to both read from a dead process, it just blocks and waits for nothing
- A side effect: `read`'s stdout is also piped to the subshell, but as the redirect is only readable, not writable, it also blocks. But as `read` never writes anything to its stdout, this does not break it.

The main idea is to block the stdin of `read` command so it waits for nothing and don't eat our stdin.

With the same idea the following functions also work
- Block output and make stdin invalid
    ```sh
    read -t "$1" <> >(:)
    ```
- Block stdin with the non-readable writer end
    ```sh
    read -t "$1" < >(:)
    ``` 

The following variants wouldn't work:
- Block only stdout, this would eat the outer stdin and early quit if encountering `^D` /`^M`
    ```sh
    read -t "$1" > >(:)
    ```
- Make stdout invalid, same as above, this would eat the outer stdin and early quit if encountering `^D` /`^M`
    ```sh
    read -t "$1" > <(:)
    ```
- Make stdin invalid, this would early quit with error (return 1) as it tries to read from non-existing fd (subshell already died)
    ```sh
    read -t "$1" < <(:)
    ```

However all of these still waste PIDs! Recall that subshells are just forked Bash processes. So the above, while should be lighter then external `sleep`, still saves no PIDs (one `fork()` and one PID for each `sleep`), and this makes your Bash script much less readable.

## Sane solution
In fact Bash does have a `sleep` built-in, although it's not compiled in but as a loadable module. On Arch Linux it's installed at `/usr/lib/bash/sleep`, you can just use the `enable` builtin to load it:
```sh
enable sleep
```
After this you can just run `sleep` similarly as the external one, and without `fork()` calls or PIDs wasted.

If you distro does not pack this, you can build `examples/loadables/sleep.c` from the source code of Bash:
```sh
curl -L https://ftp.gnu.org/gnu/bash/bash-5.3-alpha.tar.gz -o- | tar -xvz
cd bash-5.3-alpha/examples/loadables
make sleep
```
Then move `sleep` to anywhere easy to look up to and load it as follows:
```sh
enable -f /path/to/sleep/loadable
```