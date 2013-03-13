---
layout: post
title: "MogileFS读取文件的过程"
author: erning
date: 2012-05-16 10:12
comments: true
categories: [mogilefs, perl, study]
---
<!-- more -->
{% blockquote 王惠达, http://blog-wanghuida.rhcloud.com/article.php?f=mogilefs-read %}
看了几天mogilefs源码，分享一下
{% endblockquote %}

客户端读取文件代码
--------------------
>定义好domain和key就能获取文件内容

```perl
#!/usr/bin/perl

use strict;
use warnings;
use MogileFS::Client;

my %new_opt = ( 
    hosts => ["127.0.0.1:7001"],
    domain => "127.0.0.1::test",
); 
my $mogc = MogileFS::Client->new(%new_opt);

my $key = "test_key_2";
my $fd = $mogc->read_file($key);
print while<$fd>;
```

MogileFS::Client与MogileFS::Server通信
---------------
>通过get_paths取得文件的url路径 

>MogileFS::ClientHTTPFile通过HTTP获取文件句柄   

>>IO::WrapTie可以让返回的内容当文件句柄使用

```
sub read_file {
    my MogileFS::Client $self = shift;
    
    my @paths = $self->get_paths(@_);
    
    my $path = shift @paths;

    return if !$path;

    my @backup_dests = map { [ undef, $_ ] } @paths;

    return IO::WrapTie::wraptie('MogileFS::ClientHTTPFile',
                                path         => $path,
                                backup_dests => \@backup_dests,
                                readonly     => 1,
                                );
}
```
>get_paths调用MogileFS::Backend的do_request方法

```
my $res = $self->{backend}->do_request
        ("get_paths", {
            domain => $self->{domain},
            key    => $key,
            noverify => $noverify ? 1 : 0,
            zone   => $zone,
        %extra_args,
        }) or return ();

```
>MogileFS::Backend发送信息 

>消息的内容是get_paths *****

```
my MogileFS::Backend $self = shift;
my ($cmd, $args) = @_; 
my $argstr = _encode_url_string(%$args);
my $req = "$cmd $argstr\r\n";
$rv = send($sock, $req, $FLAG_NOSIGNAL);
```

服务器端接收消息
---------------------
>$code->($state);是关键，主进程监听到了事件去执行回调

```perl
# Fetch handles with read events
while (@poll) {
    my ($fd, $state) = splice(@poll, 0, 2);
    next unless $state;

    $pob = $DescriptorMap{$fd};
    if (!$pob) {
        if (my $code = $OtherFds{$fd}) {
            $code->($state);
        }    
        next;
    }    

    $pob->event_read   if $state & POLLIN && ! $pob->{closed};
    $pob->event_write  if $state & POLLOUT && ! $pob->{closed};
    $pob->event_err    if $state & POLLERR && ! $pob->{closed};
    $pob->event_hup    if $state & POLLHUP && ! $pob->{closed};
} 
```
>这就是回调函数，主进程接收请求，创建个MogileFS::Connection::Client

>MogileFS::Connection::Client会继承于Danga::Socket,会加到%DescriptorMap里,所以事件循环可以读取到

>$pob->event_read   if $state & POLLIN && ! $pob->{closed};开始读取

```
Danga::Socket->AddOtherFds( fileno($server) => sub {
    while (my $csock = $server->accept) {
        MogileFS::Connection::Client->new($csock);#也会被加到%DescriptorMap
    }   
} );
```

MogileFS::Connection::Client接收消息和处理
-------------------
>调用handle_request

>>tips:$self->{read_buf} =~ reg 取得需要的东西，并清空

```perl
# Client
sub event_read {
    my MogileFS::Connection::Client $self = shift;

    my $bref = $self->read(1024);
    return $self->close unless defined $bref;
    $self->{read_buf} .= $$bref;

    while ($self->{read_buf} =~ s/^(.*?)\r?\n//) {
        next unless length $1; 
        $self->handle_request($1);
    }   
}
```
>调用MogileFS::ProcManager->EnqueueCommandRequest来处理get_paths指令

>>tips:看到可以使用telnet 127.0.0.1 7001连接上server使用!h来获取帮助!stat可以查看服务器状态

```perl
sub handle_request {
    my ($self, $line) = @_;

    # if it's just 'help', 'h', '?', or something, do that
    #if ((substr($line, 0, 1) eq '?') || ($line eq 'help')) {
    #    MogileFS::ProcManager->SendHelp($_[1]);
    #    return;
    #}

    if ($line =~ /^!(\S+)(?:\s+(.+))?$/) {
        my ($cmd, $args) = ($1, $2);
        return $self->handle_admin_command($cmd, $args);
    }

    MogileFS::ProcManager->EnqueueCommandRequest($line, $self);
}
```
>加到待处理等列里，并开始处理

```perl
sub EnqueueCommandRequest {
    my ($class, $line, $client) = @_; 
    push @PendingQueries, [
                           $client,
                           ($client->peer_ip_string || '0.0.0.0') . " $line"
                           ];  
    MogileFS::ProcManager->ProcessQueues;
}
```
>如果有空闲的query-worker和待处理的队列，那就开始行动

>所以queryworker的数量决定了，MogileFS的处理能力，当然加大数量也会加大DB压力，memcache能缓解部分压力

>$Mappings{$worker->{fd}} = $clref;设置Mappings，等下返回结果时，这个client句柄还要用的

>$worker->write("$worker->{pid}-$worker->{reqid} $clref->[1]\r\n");向真正的worker发送消息，让他干活

>>tips:例子123-455 10.2.3.123 get_paths foo=bar&blah=bar\r\n

>$Stats都是存储当前服务器状态的，times_out_of_qworkers可以看出worker是不是处理不过来

```
sub ProcessQueues {
    return if $IsChild;

    # try to match up a client with a worker
    while (@IdleQueryWorkers && @PendingQueries) {
        # get client that isn't closed
        my $clref;
        while (!$clref && @PendingQueries) {
            $clref = shift @PendingQueries
                or next;
            if ($clref->[0]->{closed}) {
                $clref = undef;
                next;
            }
        }
        next unless $clref;

        # get worker and make sure it's not closed already
        my MogileFS::Connection::Worker $worker = shift @IdleQueryWorkers;
        if (!defined $worker || $worker->{closed}) {
            unshift @PendingQueries, $clref;
            next;
        }

        # put in mapping and send data to worker
        push @$clref, Time::HiRes::time();
        $Mappings{$worker->{fd}} = $clref;
        $Stats{queries}++;

        # increment our counter so we know what request counter this is going out
        $worker->{reqid}++;
        # so we're writing a string of the form:
        #     123-455 10.2.3.123 get_paths foo=bar&blah=bar\r\n
        $worker->write("$worker->{pid}-$worker->{reqid} $clref->[1]\r\n");

    }

    if (@PendingQueries) {
        # Don't like the name. Feel free to change if you find better.
        $Stats{times_out_of_qworkers}++;
    }
}

```

真正的QueryWorker开始干活，MogileFS::Worker::Query
----------------------
>select判断是否可以读取

>>tips:vec($rin, fileno($psock), 1) = 1; 设置2进制数值，如果fileno=5那么返回的数值就是00010000

>sysread取得socket内容

>process_line处理get_paths命令

```
sub work {
    my $self = shift;
    my $psock = $self->{psock};
    my $rin = '';
    vec($rin, fileno($psock), 1) = 1; 
    my $buf;

    while (1) {
        my $rout;
        unless (select($rout=$rin, undef, undef, 5.0)) {
            $self->still_alive;
            next;
        }    

        my $newread;
        my $rv = sysread($psock, $newread, 1024);
        if (!$rv) {
            if (defined $rv) {
                die "While reading pipe from parent, got EOF.  Parent's gone.  Quitting.\n";
            } else {
                die "Error reading pipe from parent: $!\n";
            }    
        }    
        $buf .= $newread;

        while ($buf =~ s/^(.+?)\r?\n//) {
            my $line = $1;
            if ($self->process_generic_command(\$line)) {
                $self->still_alive;  # no-op for watchdog
            } else {
                $self->validate_dbh;
                $self->process_line(\$line);
            }
        }
    }
}
```

MogileFS::Worker::Query处理命令
-------------------
process_line调用cmd_get_paths方法

```
    #设置符号表
    my $cmd_handler = *{"cmd_$cmd"}{CODE};
    $cmd_handler->($self, $args);

```
>MogileFS::Config->memcache_client 取得memcache

>my $mogfid_memkey = "mogfid:$args->{dmid}:$key";这是存fidid的key

>my $devid_memkey = "mogdevids:" . $fid->id;这是文件存在哪里的缓存key

>>tips:如果没有就查询数据库SELECT devid FROM file_on WHERE fid=?

>万事具备，就差拼url了，执行MogileFS::DevFID->new($dev, $fid); my $path = $dfid->get_url; 

>return $self->ok_line($ret);最终返回带有OK字样的结果 

>>tips:ok_line里 $self->send_to_parent("${id}${delay}OK $argline");通过ipc告诉父进程

```
sub cmd_get_paths {
    my MogileFS::Worker::Query $self = shift;
    my $args = shift;

    # memcache mappings are as follows:
    #  mogfid:<dmid>:<dkey> -> fidid
    #  mogdevids:<fidid>    -> \@devids  (and TODO: invalidate when the replication or deletion is run!)

    # if you specify 'noverify', that means a correct answer isn't needed and memcache can
    # be used.
    my $memc          = MogileFS::Config->memcache_client;
    my $get_from_memc = $memc && $args->{noverify};
    my $memcache_ttl  = MogileFS::Config->server_setting_cached("memcache_ttl") || 3600;

    # validate domain for plugins
    $args->{dmid} = $self->check_domain($args)
        or return $self->err_line('domain_not_found');

    # now invoke the plugin, abort if it tells us to
    my $rv = MogileFS::run_global_hook('cmd_get_paths', $args);
    return $self->err_line('plugin_aborted')
        if defined $rv && ! $rv;

    # validate parameters
    my $dmid = $args->{dmid};
    my $key = $args->{key} or return $self->err_line("no_key");

    # We default to returning two possible paths.
    # but the client may ask for more if they want.
    my $pathcount = $args->{pathcount} || 2;
    $pathcount = 2 if $pathcount < 2; 
 
    # get DB handle 
    my $fid; 
    my $need_fid_in_memcache = 0; 
    my $mogfid_memkey = "mogfid:$args->{dmid}:$key"; 
    if ($get_from_memc) { 
        if (my $fidid = $memc->get($mogfid_memkey)) { 
            $fid = MogileFS::FID->new($fidid); 
        } else { 
            $need_fid_in_memcache = 1; 
        } 
    } 
    unless ($fid) { 
        Mgd::get_store()->slaves_ok(sub { 
            $fid = MogileFS::FID->new_from_dmid_and_key($dmid, $key); 
        }); 
        $fid or return $self->err_line("unknown_key"); 
    } 
 
    # add to memcache, if needed.  for an hour. 
    $memc->set($mogfid_memkey, $fid->id, $memcache_ttl ) if $need_fid_in_memcache || ($memc && !$get_from_memc); 
 
    my $dmap = Mgd::device_factory()->map_by_id; 
 
    my $ret = { 
        paths => 0, 
    }; 

    # find devids that FID is on in memcache or db.
    my @fid_devids;
    my $need_devids_in_memcache = 0; 
    my $devid_memkey = "mogdevids:" . $fid->id; 
    if ($get_from_memc) { 
        if (my $list = $memc->get($devid_memkey)) { 
            @fid_devids = @$list; 
        } else { 
            $need_devids_in_memcache = 1; 
        } 
    } 
    unless (@fid_devids) { 
        Mgd::get_store()->slaves_ok(sub { 
            @fid_devids = $fid->devids; 
        }); 
        $memc->set($devid_memkey, \@fid_devids, $memcache_ttl ) if $need_devids_in_memcache || ($memc && !$get_from_memc); 
    } 
 
    my @devices = map { $dmap->{$_} } @fid_devids; 
 
    my @sorted_devs; 
    unless (MogileFS::run_global_hook('cmd_get_paths_order_devices', \@devices, \@sorted_devs)) { 
        @sorted_devs = sort_devs_by_utilization(@devices); 
    } 
 
    # keep one partially-bogus path around just in case we have nothing else to send. 
    my $backup_path; 
 
    # construct result paths 
    foreach my $dev (@sorted_devs) { 
        next unless $dev && ($dev->can_read_from);

        my $host = $dev->host;
        next unless $dev && $host; 
        my $dfid = MogileFS::DevFID->new($dev, $fid); 
        my $path = $dfid->get_url; 
        my $currently_down = 
            $host->observed_unreachable || $dev->observed_unreachable; 
 
        if ($currently_down) { 
            $backup_path = $path; 
            next; 
        } 
 
        # only verify size one first one, and never verify if they've asked not to 
        next unless 
            $ret->{paths}        || 
            $args->{noverify}    || 
            $dfid->size_matches; 
 
        my $n = ++$ret->{paths}; 
        $ret->{"path$n"} = $path; 
        last if $n == $pathcount;   # one verified, one likely seems enough for now.  time will tell. 
    } 

    # use our backup path if all else fails
    if ($backup_path && ! $ret->{paths}) {
        $ret->{paths} = 1;
        $ret->{path1} = $backup_path;
    }

    return $self->ok_line($ret);
}

sub ok_line {
    my MogileFS::Worker::Query $self = shift;

    my $delay = '';
    if ($self->{querystarttime}) {
        $delay = sprintf("%.4f ", Time::HiRes::tv_interval( $self->{querystarttime} ));
        $self->{querystarttime} = undef;
    }

    my $id = defined $self->{reqid} ? "$self->{reqid} " : '';

    my $args = shift || {};
    my $argline = join('&', map { eurl($_) . "=" . eurl($args->{$_}) } keys %$args);
    $self->send_to_parent("${id}${delay}OK $argline");
    return 1;
}

```

主进程还在事件循环，MogileFS::Connection::Worker接受刚才的OK数据
-------------------
>因为是queryworker所以调用HandleQueryWorkerResponse
```
sub event_read {
    my MogileFS::Connection::Worker $self = shift;

    # if we read data from it, it's not blocked on something else.
    $self->note_alive;

    my $bref = $self->read(1024);
    return $self->close() unless defined $bref;
    $self->{read_buf} .= $$bref;
    while ($self->{read_buf} =~ s/^(.+?)\r?\n//) {
        my $line = $1; 
        if ($self->job eq 'queryworker' && $line !~ /^(?:\:|error|debug)/) {
            MogileFS::ProcManager->HandleQueryWorkerResponse($self, $line);
        } else {
            MogileFS::ProcManager->HandleChildRequest($self, $line);
        }   
    }   
}
```

真正的返回数据
-------------------
>my ($client, $jobstr, $starttime) = @{ $Mappings{$worker->{fd}} };还记得Mappings吗，就是这里用

>$client->write("$res\r\n");写入数据，客户端接受

```
# called when we get a response from a worker.  this reenqueues the
# worker so it can handle another response as well as passes the answer
# back on to the client.
sub HandleQueryWorkerResponse {
    # got a response from a worker
    my MogileFS::Connection::Worker $worker;
    my $line;
    (undef, $worker, $line) = @_;

    return Mgd::error("ASSERT: ProcManager (Child) got worker response: $line") if $IsChild;
    return unless $worker && $Mappings{$worker->{fd}};

    # get the client we're working with (if any)
    my ($client, $jobstr, $starttime) = @{ $Mappings{$worker->{fd}} };

    # if we have no client, then we just got a standard message from
    # the queryworker and need to pass it up the line
    return MogileFS::ProcManager->HandleChildRequest($worker, $line) if !$client;

    # at this point it was a command response, but if the client has gone
    # away, just reenqueue this query worker
    return MogileFS::ProcManager->NoteIdleQueryWorker($worker) if $client->{closed};

    # <numeric id> [client-side time to complete] <response>
    my ($time, $id, $res);
    if ($line =~ /^(\d+-\d+)\s+(\-?\d+\.\d+)\s+(.+)$/) {
        # save time and response for use later
        # Note the optional negative sign in the regexp.  Somebody
        # on the mailing list was getting a time of -0.0000, causing
        # broken connections.
        ($id, $time, $res) = ($1, $2, $3);
    }

    # now, if it doesn't match
    unless ($id && $id eq "$worker->{pid}-$worker->{reqid}") {
        $id   = "<undef>" unless defined $id;
        $line = "<undef>" unless defined $line;
        $line =~ s/\n/\\n/g;
        $line =~ s/\r/\\r/g;
        Mgd::error("Worker responded with id $id (line: [$line]), but expected id $worker->{pid}-$worker->{reqid}, killing");
        $client->close('worker_mismatch');
        return MogileFS::ProcManager->AskWorkerToDie($worker);
    }

    # now time this interval and add to @RecentQueries
    my $tinterval = Time::HiRes::time() - $starttime;
    push @RecentQueries, sprintf("%s %.4f %s", $jobstr, $tinterval, $time);
    shift @RecentQueries if scalar(@RecentQueries) > 50;

    # send text to client, put worker back in queue
    $client->write("$res\r\n");
    MogileFS::ProcManager->NoteIdleQueryWorker($worker);
}

```

绕了一大圈终于回来了
-----------------
>_wait_for_readability等待socket有返回数据

>如果OK就返回decode的引用

```
# wait up to 3 seconds for the socket to come to life
unless (_wait_for_readability(fileno($sock), $self->{timeout})) {
    close($sock);
    $self->run_hook('do_request_read_timeout', $cmd, $self->{last_host_connected});
    undef $self->{sock_cache};
    return _fail("timed out after $self->{timeout}s against $self->{last_host_connected} when sending command: [$req]");
}
my $line = <$sock>;
# OK <arg_len> <response>
if ($line =~ /^OK\s+\d*\s*(\S*)/) {
    my $args = _decode_url_string($1);
    _debug("RETURN_VARS: ", $args);
    return $args;
}

```

