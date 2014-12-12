---
layout: post
date: 2014-11-03
title: Netcat "-e" Analysis
categories: linux
---

As I mentioned in a [previous post](http://blog.mark.lc/blog/2014/03/05/netcat/), netcat has this cool
`-e` parameter that lets you specify an executable to essentially turn into
a network service, that is, a process that can send and receive data over the
network. This option is option is particularly useful when called with a shell
(`/bin/sh`, `/bin/bash`, etc) as a parameter because this creates a poor man's
remote shell connection, and can also be used as a backdoor into the system.
As part of the [post-exploitation tool](https://github.com/mossberg/poet) I'm
working on, I wanted to try to add this type of remote shell feature, but it
wasn't immediately obvious to me
how something like this would be done, so I decided to dive into netcat's
source and see if I could understand how it was implemented.

Not knowing where to start, I first tried searching the file for "-e" which
brought me to:

{% highlight c %}
case 'e':           /* prog to exec */
  if (opt_exec)
ncprint(NCPRINT_ERROR | NCPRINT_EXIT,
    _("Cannot specify `-e' option double"));
  opt_exec = strdup(optarg);
  break;
{% endhighlight %}

This snippet is using the GNU argument parsing library, getopt, to check if
"-e" is set, and if not, setting the global `char*` variable `opt_exec` to the
parameter. Then I tried searching for `opt_exec`, bringing me to:


{% highlight c %}
if (netcat_mode == NETCAT_LISTEN) {
  if (opt_exec) {
ncprint(NCPRINT_VERB2, _("Passing control to the specified program"));
ncexec(&listen_sock);       /* this won't return */
  }
  core_readwrite(&listen_sock, &stdio_sock);
  debug_dv(("Listen: EXIT"));
}
{% endhighlight %}

This code checks if `opt_exec` is set, and if so calling `ncexec()`.

{% highlight c linenos %}
/* Execute an external file making its stdin/stdout/stderr the actual socket */

static void ncexec(nc_sock_t *ncsock)
{
  int saved_stderr;
  char *p;
  assert(ncsock && (ncsock->fd >= 0));

  /* save the stderr fd because we may need it later */
  saved_stderr = dup(STDERR_FILENO);

  /* duplicate the socket for the child program */
  dup2(ncsock->fd, STDIN_FILENO);   /* the precise order of fiddlage */
  close(ncsock->fd);            /* is apparently crucial; this is */
  dup2(STDIN_FILENO, STDOUT_FILENO);    /* swiped directly out of "inetd". */
  dup2(STDIN_FILENO, STDERR_FILENO);    /* also duplicate the stderr channel */

  /* change the label for the executed program */
  if ((p = strrchr(opt_exec, '/')))
    p++;            /* shorter argv[0] */
  else
    p = opt_exec;

  /* replace this process with the new one */
#ifndef USE_OLD_COMPAT
  execl("/bin/sh", p, "-c", opt_exec, NULL);
#else
  execl(opt_exec, p, NULL);
#endif
  dup2(saved_stderr, STDERR_FILENO);
  ncprint(NCPRINT_ERROR | NCPRINT_EXIT, _("Couldn't execute %s: %s"),
      opt_exec, strerror(errno));
}               /* end of ncexec() */ 
{% endhighlight %}

Here, on lines 13-16 is how the "-e" parameter really works. `dup2()` accepts
two file descriptors and after deallocating the second one (as if `close()`
was called on it), the second one's value is set to the first. So in this
case on line 13, the child process's stdin is being set to the file descriptor for the
network socket netcat opened. This means that the child process will view
any data received over the network will as input data and will act accordingly.
Then on lines 15 and 16, the stdout and stderr descriptors are also set to the
socket, which will cause any output the program has to be directed over the
network. As far as line 14 goes, I'm not sure why the original socket file descriptor
has to be closed at that exact point (and based on the comments, it seems like
the netcat author wasn't sure either).

The main point is this file descriptor swapping has essentially
converted our specified program into a network service; all the input and output
will be piped over the network, and at this point the child process can
be executed. The child will replace the netcat process and will also inherit
the newly set socket file descriptors. Note that on lines 30 and 31 there's
some error handling code that resets the original stderr for the netcat process
and prints out an error message. This is because the code should actually
never get to this point in execution due to the `execl()` call and if it does,
there was an error executing the child.

I wrote this little python program to see if I understood
things correctly:

{% highlight python %}
#!/usr/bin/env python

import sys

inp = sys.stdin.read(5)
if inp == 'hello':
    sys.stdout.write('hi\n')
else:
    sys.stdout.write('bye\n')
{% endhighlight %}

It simply reads 5 bytes from stdin and prints 'hi' if those 5 bytes were 'hello'
otherwise printing 'bye'.

Using this program as the `-e` parameter results in this:

```sh
$ netcat -e /tmp/test.py -lp 8080 &
[1] 19021
$ echo asdfg | netcat 127.0.0.1 8080
bye
[1]+  Done                    netcat -e /tmp/blah.py -lp 8080
$ netcat -e /tmp/test.py -lp 8080 &
[1] 19024
$ echo hello | netcat 127.0.0.1 8080
hi
[1]+  Done                    netcat -e /tmp/blah.py -lp 8080
```

We can see the "server" launched in the background. The `echo` command sends
data into netcat's stdin, which is being sent over the network, handled by
the python script, which sends back its response, which gets printed. Then we
can see that the server exits since the netcat process has been replaced by
the script, and the script has exited.
