isconf(8)
=========

*ISconf 4.2.8.250*\
 10/14/2014

* * * * *

-   [NAME](#toc1)
-   [SYNOPSIS](#toc2)
-   [QUICK START](#toc3)
-   [DESCRIPTION](#toc4)
-   [BACKGROUND](#toc5)
-   [PREREQUISITES](#toc6)
-   [FLAGS](#toc7)
-   [SUBCOMMANDS](#toc8)
    -   [Changing disk state](#toc9)
    -   [Branch management](#toc10)
    -   [Daemon management](#toc11)
-   [ENVIRONMENT](#toc12)
-   [FILES](#toc13)
-   [CONFIGURATION](#toc14)
-   [GLOSSARY](#toc15)
-   [INTERNALS](#toc16)
-   [BUGS/RESTRICTIONS](#toc17)
-   [SEE ALSO](#toc18)
-   [AUTHOR](#toc19)

* * * * *

NAME
====

isconf - infrastructure build and configuration manager

SYNOPSIS
========

**isconf** [**-Dhrq**] `[`**-c** *config*] `[`**-m** *message*] *verb*
[*verb\_args*] ...

QUICK START
===========

First, follow the short installation instructions in the INSTALL file
that came with this package. It's best to do this on whatever you're
using as a golden master image, then deploy that image to all of your
machines. If you're only setting up a few machines and have no image
server, then you *might* be able to get away with installing each
manually from the vendor CD, if you carefully install them each the same
way.

Later, to install the latest version of package 'foo' on ten thousand
hosts, including any hosts that are currently down or not yet built, you
can log into any host and say this:

          cd /tmp
          wget http://example.com/foo-1.2.tar.gz  
          isconf start
          isconf lock just a comment about installing foo
          isconf snap foo-1.2.tar.gz  
          isconf exec tar -xzvf foo-1.2.tar.gz 
          isconf exec make -C foo-1.2 install
          isconf exec rm -rf foo-1.2.tar.gz foo-1.2
          isconf ci

...then, on the other 9,999 hosts, run this during boot or from cron:

          isconf start
          isconf up

If you're only managing a few machines, you can probably get away with
not starting the isconf daemon at boot -- as above, just start it
manually when you need it by saying 'isconf start'. If you're turning
these machines over to someone else for long-term support, don't want to
teach them isconf, and expect them to manually make a mess anyway, then
it would make sense to leave the daemon off when you're done anyway. See
**BUGS/RESTRICTIONS** for some security reasons why it also makes sense
to leave the daemon off when you're done.

DESCRIPTION
===========

See the GLOSSARY below for terms and concepts.

ISconf can be thought of as a cross between **sudo**(8) and a
distributed version control tool like Git or Bitkeeper. Changes you make
via ISconf are journaled and added to a distributed repository, queuing
them for execution on other target machines. Those other target machines
do not need to be running, or even be built, at the time you check in
changes. As you turn on, build, reboot, and/or run 'isconf up' on other
machines, ISconf consults the journal and executes the same changes, in
the same order, on each machine.

The ISconf architecture is completely peer-to-peer; there are no central
servers or other single points of failure, and it is designed for use in
partially-partitioned networks such as DMZ environments. The
command-line client talks to a daemon which runs on each machine. The
daemon, usually started at boot, handles distributed file storage,
locking, and network communications.

ISconf is not intended for use in environments where you want to make
manual, ad-hoc, or other out-of-band changes to machines. If you don't
have the will to rebuild all of your machines from scratch so you know
what's on their disks, don't care about disaster recovery, don't need to
keep any of your machines in lock-step with each other, don't need to
test O/S changes before deploying them to production, aren't as
interested in O/S patch management, or still want to log in as root on
target machines and make arbitrary untracked changes, then you don't
want this package.

BACKGROUND
==========

One hundred years ago, automobiles were built by hand. Each vehicle was
unique, composed of parts which were often crafted on the spot. Repairs
were expensive and frequent, owners needed to be mechanics, fleetwide
engineering changes were non-existent.

Then came mass production. Today, a single automotive assembly line
produces vehicles of varying colors and options, all built from the same
basic design and tooling. Replacement parts are interchangeable;
technicians bolt engineering changes onto existing vehicles with a
reasonable expectation that the parts will fit. Economies of scale have
led to highly optimized designs, performance, and usability. Drivers
turn the key and go.

Most IT departments are nowhere near that sort of capability; they still
install and maintain operating systems and applications by hand. Each
machine is unique, reliability is elusive, users become technicians,
fixes often require re-engineering, most outages are caused by other
fixes, and infrastructure-wide changes are fraught with peril if they
are possible at all. Even basic security patches are, as a rule, applied
sporadically.

ISconf provides some of the standardized tooling needed for
deterministic, reproducible management of UNIX machines -- the kind of
reproducibility you can count on for consistency, disaster recovery,
reliability, security, and auditability. ISconf manages hosts over their
entire lifecycle following initial install, allowing you to continue to
test and deploy both major and minor changes well after the target hosts
have been placed into production service. With this tool you can safely
replace kernels and bootloaders, install new patches, packages, and
tarballs, run arbitrary commands, and even re-install the entire
operating system under program control, and do it all in a way that can
be consistently reproduced on other current or future machines.

Over the last decade, users of earlier versions of ISconf have found
that this consistency gives systems administrators enough breathing room
to "get ahead of the ticket curve", reclaim more of their nights and
weekends, and to, finally, begin to do more engineering and less
firefighting.

PREREQUISITES
=============

To do deterministic and repeatable host management, there are some
things you need to do in addition to just installing and using ISconf.
Above all, you need to maintain a reasonable level of control over the
root-owned bits which you place on your disks, both during initial
install as well as throughout their lifetime.

Automated systems administration is all about making self-modifying code
behave consistently. If you don't start from a known state and keep it
that way, then you can make no assertions about how your machines will
behave in comparison with each other -- a change which works on one host
may not work on others. Once you've destroyed this consistency, you can
no longer count on QA, disaster recovery, load balancers, distributed
applications, HA clusters, new deployments, or even single machine
rebuilds to work correctly.

1.  If two or more hosts are supposed to act the same, then you need to
    install them from the same disk image. This applies to rebuilds of a
    single host as well as multiple installs of identical hosts. See
    **base image** in the glossary.

2.  Your host install tool needs to be able to capture an image of an
    existing machine, save it on an install server, then dump that image
    onto subsequent machines verbatim, altering only those things which
    are supposed to be unique, such as IP address and hostname. Among
    Linux installers, for example, systemimager meets this requirement;
    kickstart does not. Under Solaris, you'll need to use Flash
    Archives, not Jumpstart. See **checkpoint image** in the glossary.

3.  After initial install, you need to manage hosts exclusively with
    ISconf -- no manual or other out-of-band changes. There is one
    semi-exception to this rule: You might want to use another tool to
    manage environmentally-influenced configuration files. You'll want
    to manage the binaries of that tool using ISconf, and take care to
    ensure that the external tool manages only those files which it
    must. See **environmental data** in the glossary for more
    discussion.

FLAGS
=====

Flags appear only after the isconf command name, not after subcommands
(as opposed to e.g. CVS).

**-c** *config*

Top-level configuration file. Defaults to /etc/is/main.cf.

**-D**

Show debug info on stderr.

**-m**

Message -- human-readable comment describing the change. Required only
when locking. This flag is deprecated and is likely to be removed; see
the 'lock' verb below for how to provide the message in a
forward-compatible way.

**-r**

Allow reboot if needed. Used only with the 'up' verb below. Also see the
'reboot' verb. Ordinarily you would execute 'isconf -r up' from an rc
script, which is a relatively safe time to allow reboots.

This flag has no effect unless there is a 'reboot' operation pending in
the journal. If there is a 'reboot' pending, then this flag allows the
reboot to take place. You only want to provide this flag at times when
it's safe to reboot the local machine.

Without this flag, if 'isconf up' encounters a 'reboot' operation during
journal replay, the replay stops, an error message is issued, and
subsequent changes are not applied. You'll need to run 'isconf -r up' to
continue past this point -- we cannot assume that the later changes will
work without the reboot.

**-q**

Quiet -- don't show verbose output.

**-V**

Version -- show ISconf version.

SUBCOMMANDS
===========

Subcommands are often called 'verbs' in ISconf documentation and usage.
They can be grouped into the following categories:

Changing disk state
-------------------

lock, unlock, snap, exec, reboot, ci, up

Branch management
-----------------

fork, migrate

Daemon management
-----------------

start, stop, restart

The following is a detailed description of all subcommands, in
alphabetical order. In these descriptions, the **origin** host is the
host where a user executes **lock**, **snap**, **exec**, **reboot**, or
**ci**, and the **target** host is where a user executes **up(date)**.

**ci**

Check in local changes, such as **snap** or **exec**, and release branch
lock.

Run on origin.

**exec** *command args ...*

Execute an arbitrary command. Causes the command to be executed
immediately on the local machine, and queued for execution on target
machines after **ci**.

Example:

          isconf lock "permanently shut down apache"
          isconf exec /etc/rc2.d/S85apache stop
          isconf exec rm /etc/rc2.d/S85apache 
          isconf ci

If you want to embed shell redirects or pipes in the **exec** arguments,
then you'll need to wrap the arguments in a shell invocation. For
example, this \*won't\* do what you want -- it will only change
/etc/motd on the origin machine:

          isconf exec echo web server down > /etc/motd

Here's what you really want instead:

          isconf exec sh -c "echo web server down > /etc/motd"

**fork** *newbranch*

Create a new branch from the current branch, and migrate the local host
onto the new branch. The original branch is the "parent" branch, and the
new branch is the "child" branch.

If host A executes a **fork**, then it is the only host moved to the
branch; hosts B and C do not change. If you want B or C to move to the
new branch as well, see **migrate**.

Low-level implementation: Since a journal describes the details of a
branch, then a fork essentially just copies the entire journal contents
from the parent branch into a new journal named after the child branch,
then runs the **migrate** code path.

**lock** *message* ...

Lock the branch. Required before **snap**, **exec**, **reboot**, or
**ci**, and recommended before **fork** and **migrate**. The *message*
will be recorded in the journal for each subsequent transaction until
the next **ci**.

**migrate** *branchname*

Migrates the local host onto a new branch. In human language this means
the host is going to change roles.

Switching a host to a new branch is only possible if the new branch is a
child of the host's old branch, and if there have been no transactions
executed on the host since the new branch was forked off -- in other
words, the new branch's journal content needs to be a contiguous
superset of the old branch's journal content. If these conditions aren't
met, **migrate** will exit with a non-zero return code.

**reboot**

Reboots the machine. Before reboot, adds a journal entry which will
cause all target machines on this branch to reboot at the same point in
their build. For example, this is what you might do to install and boot
a new kernel:

          isconf lock "upgrade to 2.6.20"
          isconf snap kernel-2.6.20-1.i686.rpm
          isconf exec rpm -ivh kernel-2.6.20-1.i686.rpm
          isconf reboot
          isconf ci
      
          # on other machines
          isconf -r up

Apply thought when using this verb; 'isconf up' (without the -r) won't
finish if there is a 'reboot' pending as the next action in the journal.
You need 'isconf -r up' -- and you don't want to put that in crontab,
unless you really don't mind your machines rebooting at that time. See
the **-r** flag for details.

Never say 'isconf exec reboot' -- that will only reboot the local
machine, and will never create any sort of journal entry; the reboot
kills isconf itself before the journal entry can be made. Always say
'isconf reboot' instead.

By default, ISconf runs 'shutdown -r now' to cause the reboot. If you
want or need to use a different command, see the IS\_REBOOT\_CMD
environment variable below.

**restart**

Restart the daemon. Equivalent to a **stop** followed by a **start**.

**snap** *filename*

Snapshot a file for install on target machines. Preserves the current
contents, permissions, and mode bits of the file. After **ci**, any
target host on the same branch can run 'isconf up', which will cause
ISconf to install the file on the target host.

**start**

**stop**

Start or stop the daemon.

**unlock**

Break the lock on the local branch. Use with great care. This reverses
the effect of a **lock**, invalidates the work stored in **journal.wip**
on the locking machine, and will likely require the person who set the
lock to discard their work and/or rebuild the machine where the lock was
made.

Generally speaking, it's better to pick up the telephone and call the
person who set the lock, asking them politely to finish whatever they
were doing and check it in, rather than use this subcommand.

**up**

Update. Causes the isconf daemon to attempt execution of any new
transactions in the journal. Errors and messages are copied to stderr
and stdout of **isconf** as well as to syslog. Exits with a non-zero
return code in case of error.

If used with **-r**, and if a pending **reboot** entry is encountered in
the journal, then the host will reboot.

ENVIRONMENT
===========

ISconf behavior is controlled predominantly by environment variables.
These can be set and exported before starting or restarting the isconf
daemon, or can be set in configuration files, usually **main.cf**. Any
variables set in the environment will be overridden by those set in the
configuration file.

**IS\_DOMAIN**

ISconf domain name -- more or less equivalent to an AFS cell name or a
Kerberos realm name; all of the machines sharing this name will share in
the distributed cache that makes up the ISconf repository. Normally
you'd want all of the machines in a given legal entity -- the same
corporation, for instance, to use the same domain name. This is an
arbitrary string, but by convention it is usually based on the DNS
domain name.

Rather than set this in an environment variable, you're better off
populating the **/var/is/conf/domain** file, below.

See the **domain** glossary entry.

**IS\_HOME**

The base directory which ISconf uses for data storage. Defaults to
**/var/is**.

**IS\_HMAC\_KEYS**

The name of a file which contains a list of HMAC keys. See the
**hmac\_keys** file below.

**IS\_HTTP\_PORT**

The port number which each ISconf HTTP server listens on. Used only for
file fetches between machines, and is likely to be deprecated in a
near-future release. Defaults to port 65028.

**IS\_NETS**

The name of a file which contains a list of broadcast and/or host
addresses which ISconf should advertize file updates to. See **nets**
file below. Likely to change in a future release.

**IS\_NOBROADCAST**

Boolean. If set, do not send UDP broadcast packets; only send UDP
point-to-point packets to the addresses listed in \*\*nets\* file.
Likely to change in a future release.

**IS\_PORT**

The port number which ISconf daemons use to communicate between each
other. Right now this is UDP only, but TCP will be added in 4.2.7, and
UDP is likely to be deprecated. Defaults to port 65027.

**IS\_REBOOT\_CMD**

The command which ISconf uses to reboot the machine in response to an
'isconf reboot' request. Defaults to "shutdown -r now".

FILES
=====

**/etc/is/main.cf**

Top-level configuration file for ISconf. See CONFIGURATION for details.
As of this writing, ISconf does not distribute this file for you. In
earlier versions, we used to simply rsync it from a central server at
the beginning of each execution. In a near-future version, look for it
to be managed by the distributed cache.

**/var/is**

See **IS\_HOME** above.

**/var/is/conf/domain**

Single-line file, newline optional, containing only the string which is
to be used for the ISconf domain name. See **IS\_DOMAIN** above.

**hmac\_keys**

HMAC key list, one key per line. See **IS\_HMAC\_KEYS**. If this file
exists and contains properly-formatted keys, then RFC 2104 HMAC
authentication is enabled; wire messages which are not properly
authenticated will be ignored.

The first key in the list is used for generating authentication codes on
all outgoing messages, and is the first key tried when authenticating
inbound messages. If the first key fails to authenticate an inbound
message, and if more than one key is listed in the file, then the second
and subsequent keys are tried, in order. This mechanism enables you to
update the primary key while preserving backward compatibility with
older keys, allowing for a transition period.

When updating keys, it's a good idea to first add the new key as a
secondary key to the hmac\_keys file, and deploy that to all machines.
Once you're sure that **all** of your machines (and install images) have
the new key, then move the new key up to the primary position in the
file, leaving any old key(s) in the file as secondaries, then deploy
that. Finally, once you're again sure that **all** of your machines (and
install images) are using the new primary key, then (and only then)
should you think about retiring any old key(s).

Take care when deploying this file for the first time on hosts which are
already running ISconf; those ISconf daemons which get it first will
refuse to listen to any which don't yet have the file; this will prevent
further deployment if you're using ISconf to deploy the file. To prevent
this from happening, you can include the special key **+ANY+** at the
end of the file. If encountered in the file, this special key disables
HMAC authentication of received messages, but does not prevent
generation of authentication codes on transmitted messages. What you
want to do is deploy the file with one or more real keys listed in it,
followed by the **+ANY+** key. The file might look like this when first
deployed:

          someauthenticationkey
          +ANY+
      

As you deploy the above file, hosts will begin sending authenticated
messages to each other using the **someauthenticationkey** key, but will
ignore the authentication codes they receive. Once you are sure that all
of your hosts have that copy of the file, then deploy the file again,
this time with the **+ANY+** key removed. This will cause hosts to begin
checking received authentication codes against
**someauthenticationkey**, while discarding any messages not properly
authenticated.

For best security, each key should be about 20 bytes long; see RFC 2104.
Keys can can include any ASCII character except space, newline, or the
pound (hash) (\#) sign. Lines beginning with pound signs are comments.
Blank lines are ignored. If no keys are found in the file, then the
entire file is ignored, and HMAC authentication is disabled.

ISconf checks for new versions of this file every 10 seconds when it is
processing inbound packets -- there is no need to restart the ISconf
daemon.

The hash function used internally is SHA-1, with Python's **hmac**
module doing the real work.

You should ensure that this file is only readable by root.

This entire mechanism is likely to change and/or be replaced by PGP key
signatures in a future release.

**nets**

Network broadcast list -- see **IS\_NETS** above. See t/nets for an
example. Likely to change.

CONFIGURATION
=============

ISconf uses environment variables for its configuration, and these
variables are in turn passed on to any executables ISconf calls -- see
ENVIRONMENT. These environment variables can be set in /etc/is/main.cf.
The format of this file is similar to a makefile, but whitespace is
whitespace -- tabs aren't required. Each stanza looks like this:

          target: optional includes
              var1 = value
              var2 = value

The 'target' string above is matched against the hostname; case is
significant. If it contains dots, it's matched against the FQDN. If it
starts with a caret (\^) it is a regex matched against the FQDN. The
first matching target is the only one used, however the special target
named 'DEFAULT' is always matched. Variables set in DEFAULT, earlier
includes, or earlier in the same stanza are overridden by
identically-named variables which appear later in matched stanzas.
Comments are any text following a hash (\#) on any line.

You can see the resulting environment by using the **-D** flag.

Here's an example /etc/is/main.cf:

          DEFAULT:
              NTPSERVERS = ntp1 ntp2 bigben.ucsd.edu mcs.anl.gov
              IS_NETS=/etc/is/nets
      
          NET1:
              GATEWAY = 10.10.1.1
      
          NET2:
              GATEWAY = 10.10.2.1
      
          # The host 'scotty' will end up with these environment variables
          # set during the ISconf run:
          #
          # NTPSERVERS="ntp1 ntp2 bigben.ucsd.edu mcs.anl.gov"
          # GATEWAY=10.10.1.1
          # building=23
          # floor=2
          # IS_NETS=/etc/is/nets.scotty
          #
          scotty: NET1
              building = this value is ignored
              building = 23
              floor = 2
              IS_NETS=/etc/is/nets.scotty
      
          # kirk will get:
          #
          # NTPSERVERS="ntp1 ntp2 bigben.ucsd.edu mcs.anl.gov"
          # IS_NETS=/etc/is/nets
          # GATEWAY = 10.10.2.1
          # building=52
          # floor=12
          # 
          kirk: NET2
              building = 52
              floor = 12
      
          LOST:
              building = unknown
              floor = unknown
      
          # any other host in example.com:
          #
          # NTPSERVERS="ntp1 ntp2 bigben.ucsd.edu mcs.anl.gov"
          # IS_NETS=/etc/is/nets
          # building=unknown
          # floor=unknown
          # GATEWAY=10.2.3.1
          # 
          ^.*\.example\.com: LOST
              GATEWAY = 10.2.3.1
      
          # any other host not in example.com:
          #
          # NTPSERVERS="ntp1 ntp2 bigben.ucsd.edu mcs.anl.gov"
          # IS_NETS=/etc/is/nets
          # building=unknown
          # floor=unknown
          # GATEWAY=10.0.0.1
          # 
          ^.*: LOST
              GATEWAY = 10.0.0.1

GLOSSARY
========

**base image**

An image which was created directly from vendor CD or another external
source, and which contains an empty journal. Normally as simple as
possible, with only a management tool (such as ISconf) and its
prerequisites added. See **image** glossary entry.

You will usually create only one base image per platform -- see
**one-base**. You will create at least one checkpoint image per branch.

**branch**

Host model or type. Similar usage as in software version control. A
different branch is normally used for each set of hosts that need their
own disk image and that do wildly different or conflicting things. For
example, a DNS server and a database server would tend to be on
different branches.

A branch is described by the sequence of transactions in a journal. A
new branch is created by forking an existing branch, then creating a
**checkpoint image**.

Branch names must match this regular expression:

              \w+[-\w\.]+

See also **class**.

For more discussion of what branches are, and how they contrast with
domains, see
[http://trac.t7a.org/isconf/wiki/DomainsVsBranches](http://trac.t7a.org/isconf/wiki/DomainsVsBranches).

**categories of data**

There appear to be three categories of data or executables on the disk
of a typical UNIX machine:

1.  **evolvable data** -- this includes binaries and executables
    scripts, as well as most configuration files (see glossary entry)
2.  **environmental data** -- that set of configuration data which must
    match external conditions (see glossary entry)
3.  user or business data

**checkpoint image**

An offline copy of the disk image of a given branch at a given revision,
used to differentiate branches and for speedier installs. A checkpoint
image is made by installing a host from an ancestor checkpoint or base
image, allowing its branch's journal entries to execute, then capturing
the resulting disk content. See **image** glossary entry.

**class**

This is an anti-definition: the word "class" should not be used to
describe anything related to deterministic host management. It brings
with it misconceptions, such as "hosts can be subclassed", "changes in
the parent class can be automatically and safely propagated to
subclasses", and so on; most of these misconceptions imply that *editing
history* is a safe thing to do.

**congruent**

Remaining in compliance with a fully-descriptive specification. If a
configuration management tool is congruent, the machines it manages will
remain in lock-step with the desired state. This makes it easier to
maintain a representative test environment, and allows for more
predictable disaster recovery. ISconf is congruent. Also see the
**convergent** glossary entry, and:

[http://www.infrastructures.org/papers/turing/turing.html\#methods](http://www.infrastructures.org/papers/turing/turing.html#methods)/congruence

**convergent**

Tending to converge towards a desired state. If a configuration
management tool is convergent, the machines it manages will trend
towards each other in disk state, but for practical reasons they will
rarely reach congruence. It will be difficult to maintain a
representative test environment, and changes will tend to be made first,
and tested first, in production. Predictable disaster recovery will
remain elusive. Also see the **congruent** glossary entry. For more
in-depth information about convergence, see:

[http://www.infrastructures.org/papers/turing/turing.html\#methods](http://www.infrastructures.org/papers/turing/turing.html#methods)/convergence

**domain**

An ISconf domain name is more or less equivalent to a NIS domain name,
an AFS cell name, or a Kerberos realm name. This name is an arbitrary
string, but by convention it is usually based on the DNS domain name.

ISconf domains are a security mechanism, primarily in regards to
information hiding. All of the machines sharing the same ISconf domain
name will share the same distributed cache, so root users on all of
these machines will be able to read the contents of the cache. Likewise,
machines that are in different domains will not share the same cache, so
root users of these machines will not have access to the cache contents
of the other domain. This becomes important if there is any proprietary
or sensitive information stored in the ISconf cache, for example via a
'snap' or 'exec' command.

Normally you'd want all of the machines in a given legal entity -- the
same corporation, for instance, to use the same domain name. For
example, a small company using ISconf might use an ISconf domain name of
'example.com' on all of their machines. A larger company might have
multiple divisions or subsidiaries and legal or security reasons for
segregating machines. The large campany might put most of their machines
in 'example.com', but for regulatory or security reasons might isolate a
subsidiary into 'foo.example.com', and might put their bastion and
firewall machines into 'security.example.com'. Note again that there
doens't need to be a 'security.example.com' DNS domain for this to work.

The idea of ISconf domains is to completely isolate legal entities from
each other when sharing the same net. Machines in different domains
refuse to cache each other's data, answer each other's queries, and so
on. Domains really come into play in the TCP crypto and user auth code
(ISconf 4.3 and later), where each domain has its own PGP keyring; its
own database of hosts and users, and all of the wire traffic is
encrypted accordingly.

Establishing two machines in different domains means "I don't want these
machines to ever cooperate at all. I will never merge their branches, I
don't want them to be able to share or see each other's packages, cache
space, or wire traffic."

For more discussion of what domains are, and how they contrast with
branches, see
[http://trac.t7a.org/isconf/wiki/DomainsVsBranches](http://trac.t7a.org/isconf/wiki/DomainsVsBranches).

Domain names must match this regular expression:

              \w+[-\w\.]+

**editing history**

"Editing history" is what happens when you build a machine based on a
set of instructions, then alter the instructions that you used to build
the machine. Once you've done this, there is no mathematically provable
way to ensure that your new instructions will still build the same
machine, short of building the new machine and then comparing the entire
disk content to the old one.

In ISconf, editing history would mean editing the journal file itself --
while there's nothing (currently) which would stop you from doing that,
and while the resulting file would be dutifully distributed and applied
to the target machines, it's highly discouraged and may be a lot more
difficult to do in the future, as we add things like digital signatures
and checksums to the mix.

Editing history can create major outages when:

-   you're trying to deploy changes which worked in QA (using the old
    instructions) to production (using the new instructions)
-   you're trying to execute a disaster recovery, or even a single host
    rebuild, and you no longer have the old disk content available
-   you're trying to add a new server to an existing farm and don't have
    time to resort to backups or run rsync across both disks

**environmental data**

Configuration data (usually files) whose content is predominantly
influenced by external business, political, procedural, or economic
factors, and whose function is critical to the integrity of business
data or to the operation of ISconf. Examples include files containing IP
addresses, domain names, and other information which, if out of date,
will break the ability of ISconf to continue journal replay. See also
**categories of data**.

This version of ISconf does not attempt to manage environmental data
natively. In earlier versions of ISconf, we would simply rsync
environmental configuration files (such as /etc/hosts and resolv.conf)
from a per-environment server at the beginning of each execution. We
weren't real happy with the limited flexibility that gave us, but this
method might work for you. If you want to do this, either modify or wrap
the main isconf script to call rsync, and then set up an rsync server
somewhere. See
[http://www.infrastructures.org/bootstrap/gold.shtml](http://www.infrastructures.org/bootstrap/gold.shtml)
for more details. (If demand is there, we can add an executable hook
that makes this easier.)

If a file meets the description of **evolvable data**, then it is not
environmental data, and it should be managed via a simple **isconf
snap**, rather than the means described below. For instance, /etc/passwd
and /etc/resolv.conf are usually environmental, while /etc/services and
/etc/inittab are much more influenced by local applications, and in most
cases should be managed via **isconf snap**.

A better way to manage environmental data is to store the raw data (or
pathnames pointing to the raw data) in /etc/is/main.cf and then generate
the configuration files during boot and/or cron. (Look for an isconf
verb in a near-future release which lets you export the content of
/etc/is/main.cf as a shell script. In the meantime you can do this the
other way around -- call ISconf from a wrapper script which sets up the
environment you want.)

Your goal should be to keep the set of environmental data as small as
possible, via architectural decisions in both infrastructure and
applications.

You need to be able to examine each bit of environmental data to try to
predict its behavior during deployment. Your ability to do this will
always be flawed -- you cannot possibly imagine all of the permutations
that might be encountered during future operations. Keeping the
environmental data set small reduces your workload and the risk caused
by a flawed analysis.

You need to be able to test each bit of environmental data after
deployment. Any change in environmental data, by definition, cannot be
tested anywhere except in its native environment. If this environment is
production, then we can only test these changes **after** deploying them
to production -- this is bad, but unless you have completely duplicate
networks, down to the details of IP addresses and hostnames, there's not
much you can do about it. Keeping the environmental dataset small
reduces the variations between environments; ideally, IP addresses
and/or hostnames might be the only differences you need to analyze and
test for.

The classic case of what **not** to do involves hardcoding IP addresses
in executables -- we all know this is bad, but here's why: Embedding an
IP address in a larger executable taints the entire executable,
requiring that we manage the whole file as environmental data. It's
better to move that IP address to a separate configuration file, to
shrink the size of the environmental data set.

Executables aren't the only thing that can be tainted. Embedding an IP
address into a larger configuration file of non-environmental data also
taints the rest of the configuration file. If you have ever generated
configuration files by merging IP addresses into templates of other
data, then you have experienced this case. By using templates, you
prevent taint spread.

Taken to an extreme, tainting of files and packages can cause an
explosion in the size of the environmental dataset, and an explosion of
risk, to the point where all data on disk must be considered to be
environmental, and all changes must be considered untested prior to
production rollout. If you find yourself in this situation, your best
bet might be to go with a convergent tool such as cfengine; you'll lose
congruence, though, until you're able to fix the original problems and
rebuild your machines. See **convergent** and **congruent**.

**evolvable data**

Data which can be managed via journal replay. This includes successive
versions of executables, packages, kernels, patches, and configuration
data which is not dependent on external environment. See also
**environmental data**.

Examples of evolvable data include /bin/ls, /etc/mailcap, and libc.

It's usually safe to assume that all data is evolvable until proven
otherwise. It's relatively easy to later begin managing a particular
data item as environmental data if it proves necessary.

**image**

The bits placed on disk during installation; this will be either the
base image or a checkpoint image taken from a child branch.

This version of ISconf does not do image management (it's in the release
plan). Images need to be managed and installed using a certain category
of host install tool. See **PREREQUISITES**.

**one-base**

One-base is an axiom of ISconf (and probably deterministic host
management in general) -- it says that a host of any branch can be
created by installing the base image for that platform and then
replaying that branch's journal. This means you may only need one base
image for any given platform -- starting from there you can use journal
replay to morph the image into any other image which is described by a
branch's journal.

"One base to start them all, one base to gild them, one base to boot
them all and in the darkness build them."

Sorry.

**journal**

The transaction log of all changes made to a branch, starting from the
base image. Used for replay on other hosts of the same branch.

INTERNALS
=========

The basic algorithm that ISconf uses is roughly:

-   Journal the changes that are going to be made.
-   Preserve all entries in the journal over the lifetime of the
    infrastructure.
-   Only append entries to the journal -- never delete, never alter or
    re-order.
-   Apply changes to one or more test machines by reading the journal.
-   Maintain a history of changes that have been applied to each host.
    The master copy of this history should reside on the local disk of
    that host, and must be destroyed if the disk becomes corrupt or the
    host is rebuilt.
-   Later, apply the same changes in the same order on other machines,
    by reading the same journal, using the same code path, consulting
    their local histories to see what is yet to be done.
-   (This bullet point not yet implemented in 4.2.X.) Keep track of
    those files which a human explicitly says do not need to be
    versioned, and in those cases (only), refer only to the last journal
    entry for those files. An example is resolv.conf; in this case, you
    only want the most recent version to be applied, in order to ensure
    the host will function at all. (But consider new, edited, and
    deleted configuration files; these three operations actually could
    make use of distinct handling.)

BUGS/RESTRICTIONS
=================

See
[http://trac.t7a.org/isconf/report](http://trac.t7a.org/isconf/report)
for bugs, and see notes for a given release at
[http://trac.t7a.org/isconf/roadmap?show=all.](http://trac.t7a.org/isconf/roadmap?show=all.)

This version of ISconf was assembled with the features most requested by
early adopters, and does not pretend to be secure or scalable. It is
intended for use in small deployments, trusted internal networks, and
evaluation. If you do install this version in a production environment,
you should plan to upgrade as newer versions become available.

Having said that, we do use this version of ISconf ourselves.

Because we'll need to change wire protocols to add in the security bits,
the next upgrade is likely to be a tricky procedure; you may need to
keep an old machine around for a while as a cache server until you're
sure you've upgraded all of your existing machines and updated your
checkpoint images. Keep your rollouts small for now.

Known flaws in this release include:

-   Files are transported via cleartext HTTP. Any file checked into
    ISconf is visible by anyone with a web browser. HTTP in general is a
    poor protocol for ISconf, is being used at the suggestion of an
    early adopter, and we plan to deprecate it as soon as we can get the
    consensus that it's the wrong direction.
-   Control messages are transported via UDP and/or UDP broadcast, for
    expediency. This protocol is going to be deprecated in favor of a
    TCP mesh which will do both control messages and file transport.
-   No authentication or encryption is performed for any operation on
    the wire. A properly-formatted packet can be forged to insert unsafe
    content into the journal for an entire branch. We plan to add HMAC
    soonest, and later PGP signatures and either PGP or SSL transport
    encryption as part of the TCP mesh layer.
-   Each machine stores a complete copy of all files in the cache. If
    you **snap** hundreds of megabytes of files, you will use hundreds
    of megabytes of disk space on each node. Once the TCP mesh is up,
    we'll have a protocol capable of quorum counting. This will let us
    starve the cache on ordinary nodes, while allowing designated
    "master" nodes to store a copy of everything -- the cache on these
    can then be backed up for safe-keeping as well.
-   We don't pretend to handle a certain subset of configuration files
    right now -- see the **environmental data** glossary entry.
-   Logging is rudimentary right now; everything gets dumped into
    various files in /tmp. This all needs to be migrated to syslog
    and/or files in var log.

SEE ALSO
========

  ---------------------------------------- ------------------------------------------------------------------
  Background on where all this came from   [http://www.infrastructures.org](http://www.infrastructures.org)
  ISconf main site                         [http://www.isconf.org](http://www.isconf.org)
  ISconf development site                  [http://trac.t7a.org/isconf](http://trac.t7a.org/isconf)
  cfengine(8)                              [http://www.cfengine.org](http://www.cfengine.org)
  python(1)                                [http://www.python.org](http://www.python.org)
  ---------------------------------------- ------------------------------------------------------------------

Most ISconf developers and users can be found on the infrastructures
mailing list at
[http://mailman.terraluna.org/mailman/listinfo/infrastructures](http://mailman.terraluna.org/mailman/listinfo/infrastructures)

AUTHOR
======

Steve Traugott -- [http://www.stevegt.com](http://www.stevegt.com)
