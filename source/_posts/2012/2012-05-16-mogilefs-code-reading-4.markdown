---
layout: post
title: "Perl使用Epoll优化"
author: erning
date: 2012-05-16 10:14
comments: true
categories: [mogilefs, perl, study]
---
<!-- more -->
{% blockquote 王惠达, http://blog-wanghuida.rhcloud.com/article.php?f=perl-epoll %}
看了几天mogilefs源码，分享一下
{% endblockquote %}

>优势：当有事件时才会增加到@events，无需遍历

>注意：EPOLLERR，不增加该事件，无法执行到退出

```perl
#!/usr/bin/perl

use warnings;
use Data::Dumper;
use strict;
use IO::Socket;
use POSIX;
use Socket qw(SO_KEEPALIVE);
use bytes;
use IO::Poll;
#/Developer/SDKs/MacOSX10.6.sdk/usr/include/sys/errno.h
#死活没找到，find / -name errno.h -print了下
use Errno  qw(EINPROGRESS EWOULDBLOCK EISCONN ENOTSOCK
              EPIPE EAGAIN EBADF ECONNRESET ENOPROTOOPT);

use Sys::Syscall qw(:epoll);

use constant POLLIN        => 1;
use constant POLLOUT       => 4;
use constant POLLERR       => 8;
use constant POLLHUP       => 16;
use constant POLLNVAL      => 32;


my $server = IO::Socket::INET->new(
    LocalAddr => '192.168.10.200:8888',
    Type      => SOCK_STREAM,
    Proto     => 'tcp',
    Blocking  => 0,
    Reuse     => 1,
    Listen    => 1024,  
) or die("error create socket:$! and $@\n") ;

$server->sockopt(SO_KEEPALIVE, 1);

my $epoll_test = Sys::Syscall::epoll_defined();
my $epoll = epoll_create(1024);
my @events;
my %des;

while(1){
    if(my $csock = $server->accept){
        my $fn = fileno($csock);
        epoll_ctl($epoll,EPOLL_CTL_ADD,$fn,EPOLLIN|EPOLLERR);
        $des{$fn} = $csock;
    }
    my $evcount = epoll_wait($epoll,1000,0.25,\@events);
    my $i;
    for($i = 0; $i < $evcount; $i++){
        my $ev = $events[$i];
        my $len = sysread($des{$ev->[0]},my $data,1024);
        if(!$len && $! != EWOULDBLOCK ){
            epoll_ctl($epoll,EPOLL_CTL_DEL,$ev->[0],EPOLLIN);
            delete $des{$ev->[0]};
        }else{
            print $data;
        }
    }
    #print Dumper(\%des);
    #sleep(1);
}
```

