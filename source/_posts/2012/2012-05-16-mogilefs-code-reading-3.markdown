---
layout: post
title: "Perl使用Poll多路复用"
author: erning
date: 2012-05-16 10:13
comments: true
categories: [mogilefs, perl, study]
---
<!-- more -->
{% blockquote 王惠达, http://blog-wanghuida.rhcloud.com/article.php?f=perl-poll %}
看了几天mogilefs源码，分享一下
{% endblockquote %}

>明显的缺点，需要遍历所有的socket

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

use constant POLLIN        => 1;
use constant POLLOUT       => 4;
use constant POLLERR       => 8;
use constant POLLHUP       => 16;
use constant POLLNVAL      => 32;


my $server = IO::Socket::INET->new(
    LocalAddr => '127.0.0.1:8888',
    Type      => SOCK_STREAM,
    Proto     => 'tcp',
    Blocking  => 0,
    Reuse     => 1,
    Listen    => 1024,  
) or die("error create socket:$! and $@\n") ;

$server->sockopt(SO_KEEPALIVE, 1);

my %des;

while(1){
    
    my @poll;
    while( my($fd,$sock) = each %des ){
        push @poll,$fd,POLLIN;
    }
    if(my $csock = $server->accept){
        my $fn = fileno($csock);
        push @poll,$fn,POLLIN;
        $des{$fn} = $csock;
    }
    my $count = IO::Poll::_poll(4, @poll); 
    if($count){
        while (@poll) {
            my ($fd, $state) = splice(@poll, 0, 2);
            my $len = sysread($des{$fd},my $data,1024);
            if(!$len && $! != EWOULDBLOCK ){
                #遍历所有的socket，一段时间后会监测到问题后删除
                delete $des{$fd};
            }else{
                print $data;
            }
        }
    }
    print Dumper(\%des);
    sleep(1);
}
```

