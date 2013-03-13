---
layout: post
title: "MogileFS启动流程"
author: erning
date: 2012-05-16 10:11
comments: true
categories: [mogilefs, perl, study]
---
<!-- more -->
{% blockquote 王惠达, http://blog-wanghuida.rhcloud.com/article.php?f=mogilefs-init %}
看了几天mogilefs源码，分享一下
{% endblockquote %}

实例化MogileFS::Server并运行
---------------
```perl
my $server; # server singleton
sub server {
    my ($pkg) = @_; 
    return $server ||= bless {}, $pkg;#空就创建对象，有就返回，单例
}
$server->run();
```

读取配置
---------------
>优先级，命令行>配置文件>默认配置

```
MogileFS::Config->load_config;
```

获得数据库操作句柄
---------------
>MogileFS::Store是数据库操作的基类,实际返回的对象是MogileFS::Store::MySQL

```
MogileFS::Config->check_database;
my $sto = eval { Mgd::get_store() };
sub get_store {
    return $store = MogileFS::Store->new;
}
$sto->ping;
sub ping {
    my $self = shift;
    return $self->dbh->ping;
}
$self->{dbh} = DBI->connect($self->{dsn}, $self->{user}, $self->{pass}
```

设置fidid fidid就是数据库中fid的最大值，但对这次启动来说是最小值
---------------
```
my $min_fidid = $sto->max_fidid;
sub max_fidid {
    my $self = shift;
    return $self->dbh->selectrow_array("SELECT MAX(fid) FROM file");
}
```

设置为守护进程 MogileFS::Util->daemonize()
---------------
>两次fork后成为守护进程

>>第一次fork后，父进程关闭，子进程成为孤儿进程，init接管

>>setsid让孤儿进程脱离会话，并成为进程组，脱离终端。

>>忽略挂起信号

>>第二次fork后，父进程关闭，子进程成为孤儿进程，init接管，成为守护进程

>>>据说调用第二次fork可以彻底排除取得会话的可能

```perl
daemonize() if MogileFS->config("daemonize");
sub daemonize {
    if ($pid = fork) { exit 0; }
    croak "Cannot detach from controlling terminal"
        unless $sess_id = POSIX::setsid();
    $SIG{'HUP'} = 'IGNORE';
    if ($pid = fork) { exit 0; }
    chdir "/";
    umask 0;
}
```
设置进程信号处理
---------------
>程序自己中断的信号TERM和外部中断信号INT是类似的，干掉所有子进程，删除PID文件

>管道信号忽略，(只有写，没有读的管道，会发出这种信号)

```
$SIG{TERM}  = sub {
    my @children = MogileFS::ProcManager->child_pids;
    print STDERR scalar @children, " children to kill.\n" if $DEBUG;
    my $count = kill( 'TERM' => @children );
    print STDERR "Sent SIGTERM to $count children.\n" if $DEBUG;
    MogileFS::ProcManager->remove_pidfile;
    Mgd::log('info', 'ending run due to SIGTERM');
    Sys::Syslog::closelog();

    exit 0;
};
$SIG{INT} = sub{同上}
$SIG{PIPE} = 'IGNORE';
```

创建服务器端socket
---------------
>Danga::Socket是一个socket事件驱动，处理客户端请求的是MogileFS::Connection::Client

>设置%OtherFds

```
my @servers;
foreach my $listen (@{ MogileFS->config('listen') }) {
    my $server = IO::Socket::INET->new(LocalAddr => $listen,
                                       Type      => SOCK_STREAM,
                                       Proto     => 'tcp',
                                       Blocking  => 0,
                                       Reuse     => 1,#调用Reuse可以免去服务器在终止到重启之间的所停留的时间
                                       Listen    => 1024 ) #监听队列，其实就是连接数
        or die "Error creating socket: $@\n";
    $server->sockopt(SO_KEEPALIVE, 1); 
    # save sub to accept a client
    push @servers, $server;

    Danga::Socket->AddOtherFds( fileno($server) => sub {
            while (my $csock = $server->accept) {
                MogileFS::Connection::Client->new($csock);#也会被加到%DescriptorMap
            }   
        } );
}
```

设置socket事件处理后的回调，并开始循环
---------------
>MogileFS::ProcManager  进程管理工具

>*EventLoop = *FirstTimeEventLoop; 符号表设置，调用的是FirstTimeEventLoop

```perl
MogileFS::ProcManager->push_pre_fork_cleanup(sub {
    # so children don't hold server connection open
    #关闭连接的匿名函数,给子进程用的
    close($_) foreach @servers;
});

# setup the post event loop callback to spawn jobs, and the timeout
Danga::Socket->DebugLevel(3);#始终是0
Danga::Socket->SetLoopTimeout( 250 ); #事件超时 250 milliseconds
Danga::Socket->SetPostLoopCallback(MogileFS::ProcManager->PostEventLoopChecker);#每次事件驱动完成后执行的函数
Danga::Socket->EventLoop();
```

Socket轮询器选择
---------------
>我本机为Poll,以后调用EventLoop就是PollEventLoop了,Poll需要轮询，

>线上应该为Epoll,Epoll内核提供反射模式，无需轮询

>Epoll和Poll那种更好，可能取决于活跃的Socket。。。，如果全是活跃的呢

```
sub FirstTimeEventLoop {
    my $class = shift;

    _InitPoller();

    if ($HaveEpoll) {
        EpollEventLoop($class);
    } elsif ($HaveKQueue) {
        KQueueEventLoop($class);
    } else {
        PollEventLoop($class);
    }
}
*EventLoop = *PollEventLoop;
```

第一次轮询开始
---------------
>还记得Danga::Socket->SetPostLoopCallback(MogileFS::ProcManager->PostEventLoopChecker);吗？，第一次轮询直接回调

>还记得%OtherFds吗，主进程的socket在里面,%DescriptorMap暂时还没，不过这个散列很重要

```
my @poll;
foreach my $fd ( keys %OtherFds ) {
    push @poll, $fd, POLLIN;
}    
while ( my ($fd, $sock) = each %DescriptorMap ) {
    push @poll, $fd, $sock->{event_watch};
}
my $count = IO::Poll::_poll($timeout, @poll);
unless ($count) {
    return unless PostEventLoop();
    next;
}
```

创建Monitor子进程
---------------
> 创建一副IPC通信一个给父进程，一个给子进程,并切无缓冲

```
socketpair(my $parents_ipc, my $childs_ipc, AF_UNIX, SOCK_STREAM, PF_UNSPEC )
        or die( "Sockpair failed" );
select((select( $parents_ipc ), $|++)[0]);
select((select( $childs_ipc  ), $|++)[0]);
```
>父类返回MogileFS::Connection::Worker对象,构造参数是IPC中的一个

>MogileFS::Connection::Worker继承Danga::Socket,会增加到$DescriptorMap{$fd} = $self;

>MogileFS::ProcManager->RegisterWorkerConn 设置POLLIN和$ChildrenByJob{$worker->job}->{$worker->pid} = $worker;

```
# if i'm the parent
if ($pid) {
    sigprocmask(SIG_UNBLOCK, $sigset)
        or return error("Can't unblock SIGINT for fork: $!");

    close($childs_ipc);  # unnecessary but explicit
    IO::Handle::blocking($parents_ipc, 0);

    my $worker_conn = MogileFS::Connection::Worker->new($parents_ipc);
    $worker_conn->pid($pid);
    $worker_conn->job($job);
    MogileFS::ProcManager->RegisterWorkerConn($worker_conn);
    return $worker_conn;
}
```
>子类通过job_to_class获取MogileFS::Worker::Monitor,然后实例化，调用work运作起来

>会增加到$DescriptorMap{$fd} = $self;但是不要和主进程的混淆，完全两个进程里的

>子类不会运行到exit,子进程也调用了Danga的socket事件循环

```
$_->() foreach @prefork_cleanup;
my $class = MogileFS::ProcManager->job_to_class($job)
        or die "No worker class defined for job '$job'\n";
my $worker = $class->new($childs_ipc);

# set our frontend into child mode
MogileFS::ProcManager->SetAsChild($worker);

$worker->work;
exit 0;
```

Monitor进程开始工作
-----------------
>Monitor监控数据库和Storage的情况

>最重要的是Monitor启动后，发送了消息`":monitor_just_ran"`，接着就一直循环监听和处理了

```
my $db_monitor;
$db_monitor = sub {
    $self->parent_ping;#和p保持连接
    $self->cache_refresh;#监控数据库配置,及时通知p
    $db_monitor_ran++;
    Danga::Socket->AddTimer(4, $db_monitor);#每4秒查询一次
};  

$db_monitor->();
$self->read_from_parent;

my $main_monitor;
$main_monitor = sub {
    $self->parent_ping;
    $self->usage_refresh;
    if ($db_monitor_ran) {
        $self->send_to_parent(":monitor_just_ran");
        $db_monitor_ran = 0;
    }   
    Danga::Socket->AddTimer(2.5, $main_monitor);
};  

$main_monitor->();
Danga::Socket->AddOtherFds($self->psock_fd, sub{ $self->read_from_parent }); 
Danga::Socket->EventLoop;
```

MogileFSD进程处理:monitor_just_ran消息
-------------------
>设置完需要的workers后，又和调用创建Monitor一样，创建其他的workers

```
my $child = shift;
# Gas up other workers if monitor's completed for the first time.
if (! $monitor_good) {
    MogileFS::ProcManager->set_min_workers('queryworker' => MogileFS->config('query_jobs'));
    MogileFS::ProcManager->set_min_workers('delete'      => MogileFS->config('delete_jobs'));
    MogileFS::ProcManager->set_min_workers('replicate'   => MogileFS->config('replicate_jobs'));
    MogileFS::ProcManager->set_min_workers('reaper'      => MogileFS->config('reaper_jobs'));
    MogileFS::ProcManager->set_min_workers('fsck'        => MogileFS->config('fsck_jobs'));
    MogileFS::ProcManager->set_min_workers('job_master'  => 1);
    $monitor_good = 1;
    $allkidsup    = 0;
}
```

