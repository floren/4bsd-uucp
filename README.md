# 4bsd-uucp
This is a script and a set of template files which customises a generic 4.2BSD
SimH disk image so that it acts as a uucp node and connects to other uucp
nodes via TCP links.

# Installation
You will need the *bsdtar* program installed so that tarballs compatible with 4.2BSD
can be installed. On Ubuntu, `sudo apt-get install bsdtar`. Can someone add
instructions for other systems? The source for bsdtar is at http://www.libarchive.org/

Download the SimH Github repository at https://github.com/simh/simh.
In your local copy, build a vax780 SimH binary and copy the resulting binary
somewhere useful:

```sh
make vax780
sudo cp BIN/vax780 /usr/local/bin
```

Download this Github repository. In this repository, compile the mktape
program:

```sh
cc -o mktape mktape.c
```

You will see the generic 4.2BSD SimH image, `rq.dsk.gz`. The `buildimg`
script builds a tar for each uucp system with the specific changes for that
system.

As an example, look at the `site_generate` script:

```sh
# An example of three uucp sites connected in series
#
#    site5 ----- site6 ----- site7
#
./buildimg site5:5000 site6:localhost:6000
./buildimg site6:6000 site5:localhost:5000 site7:localhost:7000
./buildimg site7:7000 site6:localhost:6000
```

*site5* allows you to telnet in on TCP port 5000. Ditto *site6* and TCP
port 6000, and *site7* and TCP port 7000.

*site5* has a (simulated) hard-wired serial connection to *site6* through
TCP port 6000. The syntax allows site6 to be on a remote computer, e.g.
available through www.somewhere.com:6000.

*site6* has a (simulated) hard-wired serial connection to *site5* and *site7*.
Finally *site7* has a (simulated) hard-wired serial connection to *site6*.
This means that each site can initiate a connection to another site; they
don't have to wait for the other site to dial in.

For each site, there are two files generated:
* `siteX.ini` is the SimH config file to run this system
* `siteX.tap` is a tarball in *tap* format with the customisations

We also need to copy the generic disk image `rq.dsk.gz` to be the disk
for each site:

```sh
zcat rq.dsk.gz | dd conv=sparse > site5.dsk
zcat rq.dsk.gz | dd conv=sparse > site6.dsk
zcat rq.dsk.gz | dd conv=sparse > site7.dsk
```

I prefer to use `dd conv=sparse` because a lot of the disk image is empty
and this saves disk space for these files.

# Customising each System

Once you have a disk, a config file and the customisation tape, here is
how you customise the system. Boot up the system:

```sh
vax780 site5.ini
```

At the `login:` prompt, login as *root* with no password. At the shell prompt,
read in and unpack the tarball, and run a script which sets appropriate
file permissions:

```sh
myname# tar xf /dev/rmt12
tar: blocksize = 1
myname# ./mkdirs
mkdir: /usr/spool/uucp: File exists
mkdir: /usr/spool/uucppublic: File exists
mkdir: C.: File exists
mkdir: D.site5X: File exists
mkdir: D.site5: File exists
mkdir: D.: File exists
mkdir: X.: File exists
mkdir: TM.: File exists
mkdir: XTMP: File exists
Now logout and login again
```

Logout (ctrl-D) and login as *root*. Your system now has a hostname:

```sh
login: root
Last login: Wed Mar  7 10:52:10 on console
You have mail.
Don't login as root, use su
site5#
```

Repeat the process for the other sites, e.g. *site6* and *site7*.

# Sending E-mail

On *site5* you can send e-mail to `site6!root` and `site6!site7!root`.
Watch out for escaping the bang characters as the shell is *csh*, e.g.

```sh
echo Hello there | mail site6\!site7\!root
```

You should be able to work out the bangpaths to send e-mails on the other
systems.

# Performing UUCP Connections

Right now, I haven't set up any cron jobs to periodically make uucp
connections, so here is how to perform a manual uucp connection.
On *site5*, to call *site6*:

```sh
# /usr/lib/uucp/uucico -r1 -ssite6 -x7
uucp site6 (3/7-10:58-177) DEBUG (ENABLED)
finds called
getto called
call: no. tty00 for sys site6
Using DIR to call
...
wanted ssword:  uucp\015\012Password:got that
send uucp
uucp site6 (3/7-10:58-177) SUCCEEDED (call to site6 )
imsg >\015\012\020<
Shere\000imsg >\020<
ROK\000msg-ROK
...
Proto started g
protocol g
uucp site6 (3/7-10:58-177) OK (startup)
...
send 41
got SY
 PROCESS: msg - SY
SNDFILE:
send 37777777621
send 37777777631
send 37777777641
rec h->cntl 42
state - 10
send 37777777651
rec h->cntl 43
...
daemon site6 (3/7-10:58-177) OK (conversation complete)
send OO 0,imsg >\020<
...
exit code 0
site5#
```

Now go to *site6* and run a similar command to forward the e-mail to
*site7*:

```sh
# /usr/lib/uucp/uucico -r1 -ssite7 -x7
```

On *site7*, you can now read your e-mail:

```sh
site7# mail
Mail version 2.18 5/19/83.  Type ? for help.
"/usr/spool/mail/root": 3 messages 3 new
>N  1 MAILER-DAEMON Sat Jul  9 23:21  35/1191 "Returned mail: Host unknown"
 N  2 MAILER-DAEMON Sat Jul  9 23:22  33/986 "Returned mail: Host unknown"
 N  3 site5!root Wed Mar  7 11:02  14/422
& 3
Message  3:
From site6!site5!root Wed Mar  7 11:02:14 1984
Received: by site7.ARPA (4.12/4.7)
        id AA00173; Wed, 7 Mar 84 11:02:14 pst
Received: by site6.ARPA (4.12/4.7)
        id AA00168; Wed, 7 Mar 84 10:58:33 pst
Received: by site5.ARPA (4.12/4.7)
        id AA00170; Wed, 7 Mar 84 10:55:27 pst
Date: Wed, 7 Mar 84 10:55:27 pst
From: site6!site5!root (Charlie Root)
Message-Id: <8403071855.AA00170@site5.ARPA>
To: site6!site7!root
Status: R

Hello there

&
```

# Automating uucp Connections
You can edit `/usr/lib/crontab` to have entries that run uucico for each site
that you connection to. Here is an example line that connects to *site6* every minute:

```sh
* * * * * /usr/lib/uucp/uucico -r1 -ssite6
```

# Testing against an External System
I've set up a simulated *seismo* at *simh.tuhs.org* port 5000. If you want to try sending
e-mail to this system, here is what you can do. In your SimH .ini file, put (or change)
this line to say:

```sh
attach dz line=0,Connect=simh.tuhs.org:5000
```

which will connect */dev/tty00* to *simh.tuhs.org* port 5000. Then in your simulated 4.2BSD
system, edit the *dialer* line in */etc/remote* to say:

```sh
dialer:dv=/dev/tty00:br#9600:
```

Now try:

```sh
# tip dialer
```

which should connect out over */dev/tty00* to *seismo* via the TCP connection.
Hit Return a few times to see if there is any response. On your host system,
do `netstat -a | grep ESTAB` and see if there is a TCP connection to
*simh.tuhs.org:5000*. To get out of tip, type in the two characters `~.`

To send mail to *seismo!root*, you need to do a few extra things. Set up your
*/usr/lib/uucp/L.sys* file with a line that says:

```sh
seismo Any;9 DIR 9600 tty00 "" "" ogin:--ogin:--ogin: uucp ssword: uucp
```

so that the uucp site *seismo* can be contacted via */dev/tty00*. Edit your
*/usr/lib/sendmail.cf* with an extra line that identifies *seismo* as a remote site:

```sh
CWseismo                 (near the other CW lines)
```

Then you can try doing:

```sh
# echo hello there | mail seismo\!root
  <wait a few seconds>
# /usr/lib/uucp/uucico -r1 -sseismo -x7
```

and you should see the debug information with parts of the uucp conversation.

# Notes and Gotchas
If you telnet into one of your sites, you will see garbage instead of
a nice `login:` prompt. This is because I had to set the DZ simulated
lines in 8-bit mode, as this is needed by uucp. However, type in `root`
and Return and you will log in.

We need to find a way in SimH to set the uucp lines in 8-bit mode but the
getty lines in 7-bit mode. Alternatively, in 4.2BSD, to set no parity on
the getty lines. Anybody have any ideas on this?

# What Next?

This all got thrown together in a couple of days, so feel free to
make suggestions or improvements. The next thing is to get C-News
and a newsreader working on these systems. Then, to get a bunch of people
to host simulated uucp sites so that we can recreate a semblance of
the uucp network that existed in the 1980s.
