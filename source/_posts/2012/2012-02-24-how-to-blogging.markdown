---
layout: post
title: "怎样维护ArchCorp的内容 (未完…)"
date: 2012-02-24 01:55
author: erning
comments: true
categories: [guide]
---

[ArchCorp][1]是采用[Octopress][2]引擎生成的纯静态网站。因此要想在此发布网志、文章或者帮助改进页面的排版和样式，首先需要了解Octopress的使用。

其次，ArchCorp的站点与我们的其他程序一样，使用Git管理并[托管][3]在[GitCorp][4]上。这样至少需要掌握Git的基本命令和GitCorp的使用。

还有文章的内容使用纯文本的标记语言[Markdown][5]，所以大家还需要了解Markdown的标记规则。

正如Octopress自己声称的“*Octopress is a blogging framework for hackers*”。面对频繁的使用命令行而不发憷是基本的要求。

## 开始之前

首先需要安装Git和Ruby(Octopress要求Ruby-1.9.2)。使用[RVM][6]或[rbenv][7]可以很方便地在机器上安装多个ruby环境。我使用的是RVM，所以本文中都以RVM为例。

如果还没有RVM和Ruby-1.9.2，请先安装。

<!-- more -->

## 设置本地环境

    $ git clone git@git.corp.anjuke.com:enzhang/arch.corp.anjuke.com.git
    $ cd arch.corp.anjuke.com

使用`ruby --version`命令，确认使用的是1.9.2版的Ruby。

接下来安装Octopress的相关依赖

    $ gem install bundler
    $ bundle install

## 本地预览

    $ rake generate
    $ rake preview

然后在浏览器中输入地址`http://localhost:4000`，就可以看到页面内容了。

## 撰写文章

文章的原始内容存在`source/_posts`目录下，以`年-月-日-标题.markdown`作为文件名。文件名将作为该篇文章的url slug使用。例如，`2012-02-24-how-to-blogging.markdown`。

想要写一篇新文章可以直接创建一个新文件，文件名应该符合上述规则。不要在文件名中使用中文、空格等URL不友好的字符。

也可以使用命令自动创建新文件。

    $ rake new_post["文章标题"]
    Creating new post: source/_posts/2012-02-24-wen-zhang-biao-ti.markdown

发现Octopress自动将“**文章标题**”转为“**wen-zhang-biao-ti**”，这真的很酷。

{% img right /medias/20120224/mvim.gif 240 214 MacVim syntax highlight %}

现在，你可以选用任何你喜欢文本编辑器来编辑刚才新创建的文件，然后就可以用浏览器来预览了。 我喜欢用[MacVim][8]，它可以很容易地加上[markdown的语法高亮][9]。 

### Meta信息

在编辑器中打开文章，可以看到前几行是这篇文章的“头信息”，处理引擎会利用这些信息来生成结果页面。例如，

```
---
layout: post
title: "怎样维护ArchCorp的内容"
date: 2012-02-24 01:55
author: erning
comments: true
categories:
published: false
---
```

我对“头信息”的建议是

 * `date` 日期与对应文件名的年月日应当一致
 * `author` 每一篇都填写完整
 * `categories` 不能使用中文，至少目前有bug

### 正文

接下来就是文章正文内容了。

> TODO:
>
>   * 常用的Markdown标记和octopress扩展
>   * 图片视频等媒体资源文件的存放规则
>  

### 保存

文章写好后，将改动提交到Git中。建议将本地的提交也push到GitCorp上的个人仓库中，这样方便其他人分享你的改进。ArchCorp的管理员可以通过这个方式，获取你的改动，合并到原始仓库，然后并发布站。

## 发布内容

就是一个Octopress站点，我们可以使用标准的[部署方法][10]，将到文章发布出去。这里两个最基本的命令是

    $ rake generate
    $ rake deploy

目前，在[ArchCorp][1]上我们不允许未经授权的发布。你需要给ArchCorp的管理员提交 ***pull-request*** ，由其将你的文章合并到官方仓库后再发布到正式站点。

未完待续...

  [1]: http://arch.corp.anjuke.com/
  [2]: http://octopress.org/
  [3]: http://git.corp.anjuke.com/cgit/enzhang/arch.corp.anjuke.com.git/
  [4]: http://git.corp.anjuke.com/
  [5]: http://github.github.com/github-flavored-markdown/
  [6]: http://rvm.beginrescueend.com/
  [7]: https://github.com/sstephenson/rbenv
  [8]: http://code.google.com/p/macvim/
  [9]: https://github.com/tpope/vim-markdown
  [10]: http://octopress.org/docs/deploying/
