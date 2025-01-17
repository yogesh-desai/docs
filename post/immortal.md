+++
date = "2010-03-25T23:34:54+02:00"
description = "man immortal(8)"
title = "immortal"

+++
⭕   `immortal` runs a command or script detached from the controlling terminal as a Unix daemon, it will supervise and restart the service if  it has been terminated.  The service can be controlled by querying a Unix socket "immortal.sock", this allows to remotely have full control over the service if required by exposing the socket using a web server like Nginx.

    immortal [-v] [-ctl dir] [-d dir] [-e dir] [-f pidfile] [-l logfile] [-logger logger] [-p child_pidfile] [-P supervisor_pidfile] [-u user] command

**`-d dir`**
Change to directory `<dir>` before starting the command

**`-e dir`**
Set environment variables specified by files within the `<dir>`.  For example, if `<dir>` is /tmp/env and contains 3 files:

    /tmp/env
        |--DEBUG  (contains true)
        |--DBHOST (contains 10.0.0.1)
        `--DBPASS (contains secret)

An environment var is going to be created for each file, in this case:

    DEBUG=true
    DBHOST=10.0.0.1
    DBPASS=secret


**`-f pidfile`**
Follow PID in pidfile.

In  some  cases  it is required to supervise applications that daemonize by default. Normally this kind of applications create a pidfile  every  time  they fork, the one can be used to follow subsequent forks and avoid creating a race condition between the supervisor trying  to  start again the process and the forks created by the application.

If using [immortalctl](/post/immortalctl) the color yellow on the  Down  column, helps  to  identify  does  process that have been forked and that currently are been supervised by the PID on the `<pidfile>`.  When the supervised application forks and creates a  `<pidfile>` a log entry (level DAEMON) will be created:

Watching pid `<int>` on file `<pidfile>`

> The  follow  pid option has better performance on Unix/BSD due the kernel event notification mechanism [kqueue(2)](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2).

**`-l logfile`**
Write stdout/stderr to `<logfile>`

**`-logger command`**
A command `<command>` to pipe stdout/stderr from stdin.  Besides writing logs locally by using the option -l, they can be sent to a remote host or by processed by another tool, for example to write logs locally and send logs remotely using logger  this  could be used:

    -l /var/log/app.log --logger "logger -h 10.0.0.1 -t app"


**`-P pidfile`**
Path to write the supervisor pidfile

**`-p pidfile`**
Path to write the child pidfile


**`-r <int>`**
Number of retries before program exit, defaults to 0 (never exit). If used
within a configuration file with the option "retries" the supervisor will not
exit and instead will send a signal "once" preventing with this starting the
process again unless the environment var [IMMORTAL_EXIT](/post/retries) is
defined which will cause to "exit" the supervisor instead.

**`-w seconds`**
Seconds to wait before starting.

**`-u user`**
Execute command on behalf `<user>`

**`-v`**
Print version


# The configuration file

**`-c <service name>.yml`**
A  configuration  file with valid YAML syntax, when using this option, it will overwrite other options, configuration format:

```yaml
cmd: <command to execute>
cwd: <change workding directory> # option -d
env:                             # option -e
    <key>: <value>
pid:
    follow: <pidfile>            # option -f
    parent: <pidfile>            # option -P
    child: <pidfile>             # option -p
log:
    file: <path>                 # option -l
    age: <int>   # seconds
    num: <int>   # int
    size: <int>  # MegaBytes
    timestamp: <bool> # prefix log with timestamp
logger: <command>                # option -logger
user: <user>                     # option -u
wait: <int>                      # option -s
require:                         # list of services needed before starting
  - foo
  - bar
```


**`-ctl /var/run/immortal/<service>`**
Path where the supervise directory will be created. This directory is unique per
service and is used to manage the service via a Unix socket besides preventing
running multiple times the same service by using a lock.

A supervise directory containing two files:

    lock
    immortal.sock

When  calling  immortal  from  the  command line, not by using immortaldir(8) as root, the supervise directory, will be created on  a hidden directory named ".immortal" within the $HOME of the user:

```sh
 ~/.immortal/<PID>
 ```


This  helps  to  run  and  supervise the same command multiple times without colliding, useful for testing or for temporary  services that will exit when server reboots.

To  keep  services  up  and running on boot time, is better to create a configuration file "run.yml" and use immortaldir(8).

# ENVIRONMENT

`IMMORTAL_SDIR` This environment variable allows to override the default supervise  directory  /var/run/immortal, used also by [immortalctl](/post/immortalctl) and [immortaldir](/post/immortaldir)

# Examples

Run command and restart it when finishes:

    immortal /bin/sh -c "sleep 5 && date > /tmp/sleep.log"

Run command, restart it when finishes and log output to file:

    immortal -l /tmp/sleep.log /bin/sh -c "date && sleep 5"

Run command, restart it when finishes, log output to file and to external
logger:

    immortal -l /tmp/sleep.log -logger "tee /tmp/sleep2.log" /bin/sh -c "date && sleep 5"

Run command, restart it when finishes, log output to file, wait 2 seconds before
start:

    immortal -s 2 -l /tmp/sleep.log /bin/sh -c "date && sleep 5"

Run a command, restart it when finishes, log output to file, and follow pid if
it forks:

    immortal -l /tmp/x.log -logger "tee /tmp/y.log" -f ./unicorn.pid  bundle exec unicorn -c unicorn.rb

Run a command, restart it when finishes, log output to file and create supervice
dir in /tmp/immortal/sleep

    immortal -l /tmp/sleep.log -ctl /tmp/immortal/sleep /bin/sh -c "sleep 5 && date"

For making `immortalctl` work using the `-ctl <dir>` the `IMMORTAL_SDIR`
environment var should be set to `/tmp/immortal`

# <a name="example"></a>Configuration example:

```yaml
cmd: bundle exec unicorn -c unicorn.rb
cwd: /test/unicorn
env:
    DEBUG: 1
    ENVIROMENT: production
pid:
    follow: /test/unicorn/unicorn.pid
    parent: /tmp/parent.pid
    child: /tmp/child.pid
log:
    file: /tmp/app.log
    age: 86400 # seconds
    num: 7     # int
    size: 1    # MegaBytes
logger: filebeat -c filebeat.yml -v -once
user: www
wait: 1
```

> Notice that when using the option `-u/user`, superuser privileges will be required


# Nginx example to manage remotely the service:

    immortal -l /tmp/sleep.log -ctl /tmp/immortal/sleep /bin/sh -c "sleep 5 && date"

Based on your shell set `IMMORTAL_SDIR`

    setenv IMMORTAL_SDIR /tmp/immortal

or

    export IMMORTAL_SDIR=/tmp/immortal

This is only required for making `immortalctl` to work, you can query directly the socket using curl, for example:

    curl --unix-socket immortal.sock http:/status -s | jq

Will output something like:

```json
{
  "pid": 7713,
  "up": "4.2s",
  "cmd": "sleep 5",
  "fpid": false,
  "count": 4
}
```

# Nginx configuration:

```nginx
upstream immortal {
    server unix:/tmp/immortal/sleep/immortal.sock;
}

server {
listen 80 default_server;
server_name _;
location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;
    proxy_http_version 1.1; # for keep-alive
    proxy_pass http://immortal/;
    proxy_redirect off;
    }
}
```

In some cases you may have to change permissions of the socket:

    chmod 766 /tmp/immortal/sleep/immortal.sock

To check the service status:

```sh
http://<domain>
```

To send signals:

```sh
http://<domain>/signal/<signal>
```

For example to stop the service:

```sh
http://<domain>/signal/stop
```

To start the service:

```sh
http://<domain>/signal/start
```

To stop the supervisor and the service:

```sh
http://<domain>/signal/halt
```

The responses are in JSON format.
