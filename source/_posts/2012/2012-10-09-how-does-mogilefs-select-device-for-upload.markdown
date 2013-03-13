---
layout: post
title: "Mogilefs设备权重算法"
date: 2012-10-09 17:55
comments: true
categories:
---

{%img right http://0.gravatar.com/avatar/78811efe9c142e0898c049686e25594d %}

### 遇到的问题

  * 有几块设备已经剩余10%容量，有几块设备剩余75%容量，为什么75%的util相当高呢
  * 原本理解mogilefs平均分配新文件到设备上，猜想应该是不对的

### 看看mogilefs到底是怎么做的呢

  * 处理文件是Query.pl干的，确认新建文件是cmd_create_open函数所为

```perl
sub cmd_create_open {
    #略

    #默认情况下设备的选择是从这开始的，参数是所有设备
    @devices = sort_devs_by_freespace(Mgd::device_factory()->get_all);

    #略
}
```

* sort_devs_by_freespace用剩余百分比做为权重，生成一个二维数组做参数[前20个]
* 真正的返回值是通过MogileFS::Util::weighted_list进行权重计算

```perl
sub sort_devs_by_freespace {
    my @devices_with_weights = map { #生成二维数组，剩余大小用来做权重
        [$_, 100 * $_->percent_free]
    } sort {  #排序，从大到小排序
        $b->percent_free <=> $a->percent_free;
    } grep {  #过滤设备，只要可以新建文件的，就是状态为alive的
        $_->should_get_new_files;
    } @_;

    my @list =
        MogileFS::Util::weighted_list(splice(@devices_with_weights, 0, 20));

    return @list;
}
```

* 这里才是算法的核心，让容量多的多放点文件，但当容量很大时基本随即，做个例子会比较直观
* 假设有两个设备，剩余的容量分别是10和75
* 那么sum就是85，如果乘以随机数的话可能是8.5(0.1)或者是76.5(0.9)
* 如果是8.5，那么75就大于8.5了，先返回了【并且这种概率大】
* 如果是76.5，那么75就不大于76.5了, 则返回另一个【这种概率小】

```perl
sub weighted_list (@) {
    my @list = grep { $_->[1] > 0 } @_; #排除掉等于0的
    my @ret; #初始化返回数组

    my $sum = 0;
    $sum += $_->[1] foreach @list; #累加剩余容量

    my $getone = sub {
        return shift(@list)->[0]
            if scalar(@list) == 1; #只有一个了，直接返回

        my $val = rand() * $sum; #总容量乘以0到1的随机数
        my $curval = 0;
        for (my $idx = 0; $idx < scalar(@list); $idx++) {
            my $item = $list[$idx];
            $curval += $item->[1]; #累加进去,保证肯定有返回
            if ($curval >= $val) { #如果curval比随机乘积大
                my ($ret) = splice(@list, $idx, 1); #切割出来
                $sum -= $item->[1]; #从总容量里减去
                return $ret->[0];   #返回这个设备
            }
        }
    };

    push @ret, $getone->() while @list; #循环执行getone，加到返回数组
    return @ret;
}
```

