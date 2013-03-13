---
layout: post
title: "进程间通信【IPC】"
author: erning
date: 2012-05-16 10:15
comments: true
categories: [mogilefs, perl, study]
---
<!-- more -->
{% blockquote 王惠达, http://blog-wanghuida.rhcloud.com/article.php?f=perl-ipc %}
看了几天mogilefs源码，分享一下
{% endblockquote %}

```perl
#!/usr/bin/perl

use warnings;
use strict;
use bytes;
use Socket;
use Data::Dumper;
use Carp;
use IO::Socket;
use Errno  qw(EINPROGRESS EWOULDBLOCK EISCONN ENOTSOCK
              EPIPE EAGAIN EBADF ECONNRESET ENOPROTOOPT);
use constant POLLIN        => 1;
use constant POLLOUT       => 4;
use constant POLLERR       => 8;
use constant POLLHUP       => 16;
use constant POLLNVAL      => 32;
use POSIX;


socketpair(my $ps,my $cs,AF_UNIX,SOCK_STREAM,PF_UNSPEC) or die ("socketpair fail");

my $pid;
die ("fork fail") unless defined ($pid = fork);

select( (select($ps),$|++)[0] );
select( (select($cs),$|++)[0] );

#组长进程
if ($pid){
    close $cs;
    undef $cs;
    our %des;

    my $fn = fileno($ps);
    $des{$fn} = $ps;

    IO::Handle::blocking($ps,0);

    while(1){
        my @poll;
        while( my($f,$s) = each %des ){
            push @poll,$f,POLLIN;
        }
        my $count = IO::Poll::_poll(4, @poll); 
        if($count){
            while (@poll) {
                my ($fd, $state) = splice(@poll, 0, 2);
                my $len = sysread($des{$fd},my $data,8) or warn("warn:$!\n");
                if(!$len && $! != EWOULDBLOCK ){
                    delete $des{$fd};
                }else{
                    print "reply:".$data;
                }
            }
        }
        #print Dumper(\%des);
        #sleep(1);
    }
}else{
    #sleep(5);
    close $ps;
    undef $ps;
    #print "$$\n";
    while(1){
        my $str = "cs msg!\n";
        my $len = length $str;
        my $rv = syswrite($cs,$str);
        #如果不sleep 服务器端瞬间拿到N多次的结果。。。 不sleep就得控制长度
        #sleep 1 if defined $rv && $rv == $len;
    }
    exit 0;
}

```

