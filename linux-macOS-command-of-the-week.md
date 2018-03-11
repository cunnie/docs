## `strace`

`strace` (as it's called on linux) is a command that displays to STDERR
the system calls that a process makes. It can be used force a program
to reveal its secrets.

A system call is a function that must be performed by the kernel
on behalf of the process, almost always involving I/O (e.g.
open(), close(), read(), write(), stat(), bind(), ...).

In this example, we use `strace` to determine where the BOSH CLI
keeps its configuration files:

```
$ bosh envs
bosh envs
URL                  Alias
bosh.run.pivotal.io  prod
 # where does BOSH CLI keep its list of environments it knows about?
strace -f bosh envs 2> /tmp/junk.strace > /dev/null
 # `-f` means "follow": trace the process and also (follow) its child processes
grep open /tmp/junk.strace
 # we grep for "open" because that's the system call a process makes
 # in order to read from a file
...
[pid 25210] openat(AT_FDCWD, "/home/bcunnie/.bosh/config", O_RDONLY|O_CLOEXEC) = 4
```

Aha! the BOSH CLI keeps its configuration files in `~/.bosh/config`.

On macOS the equivalent program is `dtruss`, but you need to be root to run it
and it doesn't seem to work as reliably as Linux's `strace`. On Solaris the
equivalent is named `truss`, and on FreeBSD it's `ktrace`.

## Who is attacking me? China!

You can find out who is attacking me by examining logs and using various
web-based lookup tools.

Let's ssh onto the jumpbox. We'll need to be root because the logs we will be
examining are only readable by root. That means we'll need to ssh in as user
`ubuntu` (which has `sudo`)

We have our IP address: **123.124.233.43**

Go to <https://www.arin.net/>, put that IP address in whois.

Aha! It's registered to APNIC (Asia Pacific). Let's try the URL they
give us: <http://wq.apnic.net/apnic-bin/whois.pl>

We found out more: the IP Address is from _China Unicom Beijing Province Network_, but they've been allocated a /12 (123.112.0.0/12), and the
word "Network" appears in their name, so it's most likely one of their
customers. We can report abuse, or we can let it slide.

## How to make a 5 TiB file that takes up no space

What you'll learn:

- sparse files
- stat: system call `man 2 stat` and command `man 1 stat`,
  inodes
- `dd`

Google interview question:

> What values are returned from the stat() system call?

Let's make a big file (don't try this on macOS):

```
dd if=/dev/null of=/tmp/5TiB.bin bs=5120k seek=1024k count=0
ls -lh /tmp/5TiB.bin
  -rw-rw-r-- 1 cunnie cunnie 5.0T Mar  1 09:12 /tmp/5TiB.bin
stat /tmp/5TiB.bin
  File: /tmp/5TiB.bin
  Size: 5497558138880	Blocks: 0          IO Block: 4096   regular file
  Device: 801h/2049d	Inode: 1048717     Links: 1
  Access: (0664/-rw-rw-r--)  Uid: ( 1000/  cunnie)   Gid: ( 1000/  cunnie)
  Access: 2018-03-01 09:10:29.717549491 -0800
  Modify: 2018-03-01 09:12:50.467192106 -0800
  Change: 2018-03-01 09:12:50.467192106 -0800
  Birth: -
```
