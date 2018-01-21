---
layout: post
title: Simple Linux Daemon Tutorial
date: 2018-01-06 00:00:00 +0300
img: standing-daemon.jpg
tags: [Linux, daemon, sysAdmin, C, tutorial]
---
As you might know daemons are just processes that lurk in the background
and do some kind of work when needed without the user explicitly knowing,
that they exists.
In this post I will show you how to write a simple daemon which
executes ```who``` command regularly,
logs it's output to a specific file using syslog, has a regular
logrotate and is automatically started upon system boot.

As daemons are just some programs we can write them many programming languages,
most linux users prefer BASH or C. In this tutorial I will use C.

The typical stucture of a daemon program is split in two parts, the first
is the initial setup (forking processes, detaching from parent etc). And the second
is the actual functionality of the program.

So the first thing we need to do is detach this process from the parent so that if, the
shell that ran the program is closed the daemon continues to live on. You can do it
manually by using forks but there is actually a ```daemon``` function which does this
the proper way so we need to just use it. The other thing that is a good practice is to
save the PID of the daemon to some file. We can make the program take command line argument
which will be the path to the file where we want to write the PID of the daemon. The setup
part of the our **example** daemon looks like this.
```c
// example.c
int status = daemon(0, 0);
if (status == -1)
{
    exit(5);
}

char* path_to_pid_file;
if (argc == 2)
{
    path_to_pid_file = argv[1];
    FILE* fd = fopen(path_to_pid_file, "w");
    if (fd == NULL)
    {
        exit(6);
    }
    fprintf(fd, "%d\n", getpid());
    fclose(fd);
}
```

The second part of the daemon is where the actual logic of it resides, most deamons
comunicate with other processes and this is done by sockets but in our case we will just
log the output of the ```who``` command every 5 sec.
```c
// example.c
FILE* pipeHandle;
const char* command = "who";
char data[1024];

while (1)
{
    pipeHandle = popen(command, "r");
    while (fgets(data, 1024, pipeHandle) != NULL)
    {
        syslog(LOG_MAKEPRI(LOG_LOCAL7, LOG_INFO), "%s", data);
    }

    pclose(pipeHandle);
    sleep(5);
}
```
As you can see we just get the output of ```who``` and log it using the ```syslog```
utility. In our case we log to the **Local7** facility with priority **Info**.
But in order for our logging to work we need to add a configuration to **/etc/rsyslog.conf**.
The line is ```local7.info /var/log/local7.info```. This tells the syslog utility to log the
specified facility and priority to **local7.info** file, which we have created.

After doing this we have a daemon which logs its output regularly. Since daemons tend to be
long running we need to use the ```logrotate``` system tool. Logrotate is a utility that
compresses/removes old logs so that your machine does not run out of disk space. It is usually
ran as a daily **cronjob**.

To make this working we need to add a logrotate configuration file for our daemon. The place to do so
is **/etc/logrotate.d/my-daemon-logrotate-config-file**
```
#my-daemon-logrotate-config-file
/var/log/local7.info {
	rotate 3
	daily
	missingok
	notifempty
	delaycompress
	compress
	postrotate
		invoke-rc.d rsyslog rotate > /dev/null
	endscript
}
```
You can read about logrotat in the man page but the gist of the given cofinguration
is that we will keep 3 logs, the rotation will be daily and we will compress the logs.

The last thing we need to do is to make an init script. This tutorial is for SysV
init style systems. We need to place a shell script in **/etc/init.d**. The script should
support command line arguments **start/restart/stop**, usually there are a few others but we'll keep
it simple.
```bash
if [ "$1" == "start" ]
then
	/usr/bin/exampled /run/exampled.pid
elif [ "$1" == "restart" ]
then
	pkill exampled
	/usr/bin/exampled /run/exampled.pid
elif [ "$1" == "stop" ]
then
	pkill exampled
fi
```
The script above assumes that you've compiled the daemon program with name **exampled**
and placed it int **/usr/bin**

After we have the init scrip ready there is one last thing we need to do. And that is
to make the script run with start argument when the system boots. To do this
add symlink in **/etc/rc5.d** with name **S99example** which points to the init script
we've just crated.
With this setup, when the system boots on run-level 5 it will execute the
init scrip with start argument.

After doing this we have a functional linux daemon, which logs
and logrotates it's messages and is automatically started upon boot. The
complete code can be found at [my github](https://github.com/Pavaka/Playground/tree/master/simple-daemon).