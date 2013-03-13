---
layout: post
title: "Use Supervisord with Storm and ZooKeeper"
date: 2012-08-27 21:25
author: jizhang
comments: true
categories: [python]
---

{%img right http://www.gravatar.com/avatar/b881be65f3c4f45dd68bcf0fbe6ba82b.png %}

Since both 'bin/storm (nimbus|supervisor|ui)' and 'bin/zkServer.sh start' tend to fork a new process to start the JVM, it's difficult to use 'supervisorctl stop ...' to control the processes.

There're two solutions:

1. Use supervisord 3.0a13 (current version is 3.0a12), which provides two options 'stopasgroup, killasgroup' in [program:x] section, making it possible to stop or kill the process group. But the new release doesn't seem to come up soon.

2. Configure the full command to supervisord. bin/storm and bin/zkServer.sh are both control scripts. The essential part is the JVM command. For Storm, when we startup one the components (nimbus, etc.), it'll print the full command. For ZooKeeper, use 'bin/zkServer.sh print-cmd'. Remember to check whether the command includes the full path to the binaries. If not, use the 'directory' option in [program:x] section.

Here is a sample configuration of supervisord.conf:

```
[supervisord]
logfile = /tmp/supervisord.log

[supervisorctl]
serverurl = unix:///tmp/supervisor.sock

[unix_http_server]
file = /tmp/supervisor.sock

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[program:zookeeper]
command = /usr/lib/jvm/java-1.7.0-openjdk.x86_64/bin/java -Dzookeeper.log.dir="." ...
stdout_logfile = /home/jizhang/Applications/zookeeper/zookeeper.out
stderr_logfile = /home/jizhang/Applications/zookeeper/zookeeper.err
autorestart = true

[program:storm-nimbus]
command = java -server -Dstorm.home=/home/jizhang/Applications/storm -Dstorm.options= -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib...
autorestart = true

[program:storm-superviosr]
command = java -server -Dstorm.home=/home/jizhang/Applications/storm -Dstorm.options= -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib ...
autorestart = true

[program:storm-ui]
command = java -server -Dstorm.home=/home/jizhang/Applications/storm -Dstorm.options= -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib ...
autorestart = true
```