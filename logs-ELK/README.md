# Rsyslog Omelasticsearch/Logstash Elasticsearch Kibana Stack

In this example nginx logs are being used. You can find all necessary configs in this repo.

Few information about elasticsearch:

- elasticsearch creates indices automaticly when omelasticsearch rsyslog plugin talks to elasticsearch
- /etc/elasticsearch/jvm.options - definition of heap size of JVM

Logstash similar to the omelasticsearch plugin automaticly creates indices in elasticsearch.

## Usefull commands

`rsyslogd -N1 # Validate rsyslog config`

`nginx -t # Validate nginx config`

`sudo rsyslogd -dn #Run rsyslog daemon in frontend and in debug mode`

`journalctl # Shows all the logs`

`logger -p kern.notice "Testing Log Entry" # Create kern.notice log with a speciied message. It shows in journalctl`

`sudo tcpdump -s 10240 -A -i any -p -n -s 1500 "tcp port 9200" # Debug issues between rsyslog and elasticsearch`

`curl 'localhost:9200/_cat/indices?v' # Show indices in elasticsearch`

`curl -XDELETE localhost:9200/logstash-2017.01.1* # Delete indices which fullfil pattern "logstash-2017.01.1*"`


`curl -XDELETE 'http://localhost:9200/*' # Delete all indices`

For some, the ability to delete all your data with a single command is a very scary prospect. If you want to eliminate the possibility of an accidental mass-deletion, you can set the following to true in your elasticsearch.yml:

`action.destructive_requires_name: true`

## Usefull links

[`https://sematext.com/blog/2013/07/01/recipe-rsyslog-elasticsearch-kibana/`](https://sematext.com/blog/2013/07/01/recipe-rsyslog-elasticsearch-kibana/)

[`https://www.digitalocean.com/community/tutorials/how-to-centralize-logs-with-rsyslog-logstash-and-elasticsearch-on-ubuntu-14-04`](https://www.digitalocean.com/community/tutorials/how-to-centralize-logs-with-rsyslog-logstash-and-elasticsearch-on-ubuntu-14-04)

[`https://timothy-quinn.com/running-the-elk-stack-on-centos-7-and-using-beats/`](https://timothy-quinn.com/running-the-elk-stack-on-centos-7-and-using-beats/)

[`https://medium.com/@thomasdecaux/exploit-nginx-access-log-with-rsyslog-logstash-elasticsearch-and-kibana-48ab5c71b42d#.fkw43l1gu`](https://medium.com/@thomasdecaux/exploit-nginx-access-log-with-rsyslog-logstash-elasticsearch-and-kibana-48ab5c71b42d#.fkw43l1gu)

## Good articles and their sources explaining how does the logging work in linux:

### [http://unix.stackexchange.com/questions/205883/understand-logging-in-linux](http://unix.stackexchange.com/questions/205883/understand-logging-in-linux)

The old way how logging on linux was working:

```
Simplified, it goes more or less like this:

The kernel logs messages (using the printk() function) to a ring buffer in kernel space. These messages are made available to user-space applications in two ways: via the /proc/kmsg file (provided that /proc is mounted), and via the sys_syslog syscall.

There are two main applications that read (and, to some extent, can control) the kernel's ring buffer: dmesg(1) and klogd(8). The former is intended to be run on demand by users, to print the contents of the ring buffer. The latter is a daemon that reads the messages from /proc/kmsg (or calls sys_syslog, if /proc is not mounted) and sends them to syslogd(8), or to the console. That covers the kernel side.

In user space, there's syslogd(8). This is a daemon that listens on a number of UNIX domain sockets (mainly /dev/log, but others can be configured too), and optionally to the UDP port 514 for messages. It also receives messages from klogd(8) (syslogd(8) doesn't care about /proc/kmsg). It then writes these messages to some files in /log, or to named pipes, or sends them to some remote hosts (via the syslog protocol, on UDP port 514), as configured in /etc/syslog.conf.

User-space applications normally use the libc function syslog(3) to log messages. libc sends these messages to the UNIX domain socket /dev/log (where they are read by syslogd(8)), but if an application is chroot(2)-ed the messages might end up being written to other sockets, f.i. to /var/named/dev/log. It is, of course, essential for the applications sending these logs and syslogd(8) to agree on the location of these sockets. For these reason syslogd(8) can be configured to listen to additional sockets aside from the standard /dev/log.

Finally, the syslog protocol is just a datagram protocol. Nothing stops an application from sending syslog datagrams to any UNIX domain socket (provided that its credentials allows it to open the socket), bypassing the syslog(3) function in libc completely. If the datagrams are correctly formatted syslogd(8) can use them as if the messages were sent through syslog(3).

Of course, the above covers only the "classic" logging theory. Other daemons (such as rsyslog and syslog-ng, as you mention) can replace the plain syslogd(8), and do all sorts of nifty things, like send messages to remote hosts via encrypted TCP connections, provide high resolution timestamps, and so on. And there's also systemd, that is slowly phagocytosing the UNIX part of Linux. systemd has its own logging mechanisms, but that story would have to be told by somebody else. :)

Differences with the *BSD world:

On *BSD there is no klogd(8), and /proc either doesn't exist (on OpenBSD) or is mostly obsolete (on FreeBSD and NetBSD). syslogd(8) reads kernel messages from the character device /dev/klog, and dmesg(1) uses /dev/kmem to decode kernel names. Only OpenBSD has a /dev/log. FreeBSD uses two UNIX domain sockets /var/run/log and var/rub/logpriv instead, and NetBSD has a /var/run/log.
```


How nowadays does the logging work on linux:

```
The other answer explains, as its author says, "classic logging" in Linux. That's not how things work in a lot of systems nowadays.
The kernel

The kernel mechanisms have changed.

The kernel generates output to an in-memory buffer. Application softwares can access this in two ways. The logging subsystem usually accesses it as a pseudo-FIFO named /proc/kmsg. This source of log information cannot usefully be shared amongst log readers, because it is read-once. If multiple processes share it, they each get only a part of the kernel log data stream. It is also read-only.

The other way to access it is the newer /dev/kmsg character device. This is a read-write interface that is shareable amongst multiple client processes. If multiple processes share it, they all read the same complete data stream, unaffected by one another. If they open it for write access, they can also inject messages into the kernel's log stream, as if they were generated by the kernel.

/proc/kmsg and /dev/kmsg provide log data in a non-RFC-5424 form.

Applications

Applications have changed.

The GNU C library's syslog() function in the main attempts to connect to an AF_LOCAL datagram socket named /dev/log and write log entries to it. (The BSD C library's syslog() function nowadays uses /var/run/log as the socket name, and tries /var/run/logpriv first.) Applications can of course have their own code to do this directly. The library function is just code (to open, connect, write to, and close a socket) executing in the application's own process context, after all.

Applications can also send RFC 5424 messages via UDP to a local RFC 5426 server, if one is listening on an AF_INET/AF_INET6 datagram socket on the machine.

Thanks to pressure from the daemontools world over the past two decades, a lot of dæmons support running in a mode where they don't use the GNU syslog() C library function, or UDP sockets, but just spit their log data out to standard error in the ordinary Unix fashion.

####log management with nosh and the daemontools family in general

With the daemontools family of toolsets there's a lot of flexibility in logging. But in general across the whole family the idea is that each "main" dæmon has an associated "logging" dæmon. "main" dæmons work just like non-dæmon processes and write their log messages to standard error (or standard output), which the service management subsystem arranges to have connected via a pipe (which it holds open so that log data are not lost over a service restart) to the standard input of the "logging" dæmon.

All of the "logging" dæmons run a program that logs somewhere. Generally this program is something like multilog or cyclog that reads from its standard input and writes (nanosecond timestamped) log files in a strictly size-capped, automatically rotated, exclusive-write, directory. Generally, too, these dæmons all run under the aegises of individual dedicated unprivileged user accounts.

So one ends up with a largely distributed logging system, with each service's log data processed separately.

One can run something like klogd or syslogd or rsyslogd under a daemontools-family service management. But the daemontools world realized many years ago that the service management structure with "logging" dæmons lends itself quite neatly to doing things in a simpler fashion. There's no need to fan all of the log streams into one giant mish-mash, parse the log data, and then fan the streams back out to separate log files; and then (in some cases) bolt an unreliable external log rotation mechanism on the side. The daemontools-family structure as part of its standard log management already does the log rotation, the logfile writing, and the stream separation.

Furthermore: The chain-loading model of dropping privileges with tools common across all services means that the logging programs have no need of superuser privileges; and the UCSPI model means that they only need to care about differences such as stream versus datagram transports.

The nosh toolset exemplifies this. Whilst one can run rsyslogd under it, out of the box, and just manage kernel, /run/log, and UDP log input in the old way; it also provides more "daemontools native" ways of logging these things:

- a klogd service that reads from /proc/kmsg and simply writes that log stream to its standard error. This is done by a simple program named klog-read. The associated logging dæmon feeds the log stream on its standard input into a /var/log/sv/klogd log directory.
- a local-syslog-read service that reads datagrams from /dev/log (/run/log on the BSDs) and simply writes that log stream to its standard error. This is done by a program named syslog-read. The associated logging dæmon feeds the log stream on its standard input into a /var/log/sv/local-syslog-read log directory.
- a udp-syslog-read service that listens on the UDP syslog port, reads what it is sent to it and simply writes that log stream to its standard error. Again, the program is syslog-read. The associated logging dæmon feeds the log stream on its standard input into a /var/log/sv/udp-syslog-read log directory.
- (on the BSDs) a local-priv-syslog-read service that reads datagrams from /run/logpriv and simply writes that log stream to its standard error. Again, the program is syslog-read. The associated logging dæmon feeds the log stream on its standard input into a /var/log/sv/local-priv-syslog-read log directory.

The toolset also comes with an export-to-rsyslog tool that can monitor one or several log directories (using a system of non-intrusive log cursors) and send new entries in RFC 5424 form over the network to a designated RFC 5426 server.

#### log management with systemd

systemd has a single monolithic log management program, systemd-journald. This runs as a service managed by systemd.

- It reads /dev/kmsg for kernel log data.
- It reads /dev/log (a symbolic link to /run/systemd/journal/dev-log) for application log data from the GNU C library's syslog() function.
- It listens on the AF_LOCAL stream socket at /run/systemd/journal/stdout for log data coming from systemd-managed services.
- It listens on the AF_LOCAL datagram socket at /run/systemd/journal/socket for log data coming from programs that speak the systemd-specific journal protocol (i.e. sd_journal_sendv() et al.).
- It mixes these all together.
- It writes to a set of system-wide and per-user journal files, in /run/log/journal/ or /var/log/journal/.
- If it can connect (as a client) to an AF_LOCAL datagram socket at /run/systemd/journal/syslog it writes journal data there, if forwarding to syslog is configured.
- If configured, it writes journal data to the kernel buffer using the writable /dev/kmsg mechanism.
- If configured, it writes journal data to terminals and the console device as well.

Bad things happen system-wide if this program crashes, or the service is stopped.

systemd itself arranges for the standard outputs and errors of (some) services to be attached to the /run/systemd/journal/stdout socket. So dæmons that log to standard error in the normal fashion have their output sent to the journal.

This completely supplants klogd, syslogd, syslog-ng, and rsyslogd.

These are now required to be systemd-specific. On a systemd system they don't get to be the server end of /dev/log. Instead, they take one of two approaches:

- They get to be the server end of /run/systemd/journal/syslog, which (if you remember) systemd-journald attempts to connect and write journal data to. A couple of years ago, one would have configured rsyslogd's imuxsock input method to do this.
- They read directly from the systemd journal, using a systemd-specific library that understands the binary journal format and that can monitor the journal files and directory for new entries being added. Nowadays, one configures rsyslogd's imjournal input method to do this.

```
### [http://unix.stackexchange.com/questions/198178/what-are-the-concepts-of-kernel-ring-buffer-user-level-log-level](http://unix.stackexchange.com/questions/198178/what-are-the-concepts-of-kernel-ring-buffer-user-level-log-level)

```
Yes, all of this has to do with logging. No, none of it has to do with runlevel or "protection ring".

The kernel keeps its logs in a ring buffer. The main reason for this is so that the logs from the system startup get saved until the syslog daemon gets a chance to start up and collect them. Otherwise there would be no record of any logs prior to the startup of the syslog daemon. The contents of that ring buffer can be seen at any time using the dmesg command, and its contents are also saved to /var/log/dmesg just as the syslog daemon is starting up.

All logs that do not come from the kernel are sent as they are generated to the syslog daemon so they are not kept in any buffers. The kernel logs are also picked up by the syslog daemon as they are generated but they also continue to be saved (unnecessarily, arguably) to the ring buffer.

The log levels can be seen documented in the syslog(3) manpage and are as follows:

- LOG_EMERG: system is unusable
- LOG_ALERT: action must be taken immediately
- LOG_CRIT: critical conditions
- LOG_ERR: error conditions
- LOG_WARNING: warning conditions
- LOG_NOTICE: normal, but significant, condition
- LOG_INFO: informational message
- LOG_DEBUG: debug-level message

Each level is designed to be less "important" than the previous one. A log file that records logs at one level will also record logs at all of the more important levels too.

The difference between /var/log/kern.log and /var/log/mail.log (for example) is not to do with the level but with the facility, or category. The categories are also documented on the manpage.

```


