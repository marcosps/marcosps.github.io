+++
title = "NO_NEW_PRIVS: avoiding privilege escalation"
date = "2018-05-22"
description = "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
tags = [
    "no_new_privs",
    "execve",
    "linux",
]
+++

Proposed in [2012](https://lwn.net/Articles/475362), the [*NO_NEW_PRIVS*](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt) flag made possible to any process to avoid privilege escalation when this behavior is not desired. After the flag is set, it persists across [execve](http://man7.org/linux/man-pages/man2/execve.2.html), [clone](http://man7.org/linux/man-pages/man2/clone.2.html) and [fork](http://man7.org/linux/man-pages/man2/fork.2.html) syscalls, and cannot be cleared. This can help you to avoid exploitation of vulnerable software, since the attacker will be running as an ordinary user.

The *NO_NEW_PRIVS* flag is already beeng used by some projects that try to make the running environment more secure, specially container engines and sandbox applications. Some examples are [Docker](https://www.projectatomic.io/blog/2016/03/no-new-privs-docker), [Bullewrap](https://github.com/projectatomic/bubblewrap), and [Firejail](https://github.com/netblue30/firejail).

There are cases where privilege escalation is necessary, for example, to execute a small task that can’t be done by an unprivileged user. This can be achieved by creating a new binary, that only do a very specific task, and have the [setuid](https://en.wikipedia.org/wiki/Setuid) bit set (which change the current uid by the owner of the binary) or file [capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)(which can hold *CAP_SYS_ADMIN* for example, and so the current uid becomes practically root).

Another important note for *NO_NEW_PRIVS* is, after this flag is set, an unprivileged process can install [seccomp_filters](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt).

As described by the official kernel documentation about *NO_NEW_PRIVS*, this flag is set by using [prctl](http://man7.org/linux/man-pages/man2/prctl.2.html), as exemplified below:

`prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);`

Let’s check how this works. First, we create a new binary called *caller*, which will be responsible for executing another binary, simply called *getuid*. The second binary will just print the current [effective user](https://en.wikipedia.org/wiki/User_identifier#Effective_user_ID). Let’s take a look in both binaries, starting from *caller*:

```
#include <string.h>
#include <sys/prctl.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	if (argc != 3)
		errx(0, "Usage: %s <0|1> <path to binary>", argv[0]);
	if (!strncmp(argv[1], "1", 1) &&
	    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) == -1)
		errx(1, "no_new_privs failed");

	execlp(argv[2], argv[2], NULL);
	err(1, "execlp");
}
```

The first binary, *caller*, will receive two parameters, the first one specifies if the user wants to set the *NO_NEW_PRIVS* flag, and the second one receives a path to the binary we want to execute. After compiling getuid, we need to change the owner, and turn on the [setuid](https://en.wikipedia.org/wiki/Setuid) bit of the resulting binary:

```
$ gcc getuid.c -o getuid
$ sudo chown root:root getuid
$ sudo chmod +s getuid
```

Now, let’s make use of both binaries. Let’s assume you have them in the same directory. First, without setting *NO_NEW_PRIVS*:

```
$ ./caller 0 ./getuid
euid: 0
```

As expected, the printed effective user id is 0, meaning that we are root, thanks to the setuid bit being set and the owner of the binary being root. What happens when we turn on the *NO_NEW_PRIVS* flag in *caller*?

```
$ ./caller 1 ./getuid
euid: 1000
```

As expected, the effective user id is the one who executes *caller*, so, no privileges were escalated.

We can also exemplify this behavior using [setpriv](http://man7.org/linux/man-pages/man1/setpriv.1.html), which is part of [util-linux](https://en.wikipedia.org/wiki/Util-linux), to test the *NO_NEW_PRIVS* flag. Take a look below:

```
$ setpriv ./getuid 
euid: 0
$ setpriv --no-new-privs ./getuid 
euid: 1000
```

As you can see, the output is the same from the *caller*, as it uses the same feature to avoid privilege escalation.

So, the general suggestion is: always set *NO_NEW_PRIVS* whenever you don’t need “new privileges” to be added to your process.

See you next time!

