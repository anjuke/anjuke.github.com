---
layout: post
title: "zc.buildout Workflow"
date: 2012-10-02 14:44
author: jizhang
comments: true
categories: [python]
---

{%img right http://www.gravatar.com/avatar/b881be65f3c4f45dd68bcf0fbe6ba82b.png %}

This article aims to provide a concrete workflow of using zc.buildout to deploy development and production environment.

Project Structure (BuildOut SKELeton)
-------------------------------------

Here is a minimum application structure with zc.buildout.

```bash
$ git clone git@github.com:jizhang/boskel.git
$ find boskel
boskel
boskel/.gitignore
boskel/bootstrap.py
boskel/buildout.cfg
boskel/offline.cfg
boskel/setup.py
boskel/src
boskel/src/boskel
boskel/src/boskel/__init__.py
```

Development Environment
-----------------------

To initiate the application:

```bash
$ cd boskel
$ python bootstrap.py --distribute
$ bin/buildout
```

Now, a fully functional and isolated python environment has been built. Try the following commands:

```bash
$ bin/python
>>> import boskel
>>>
```

It shows the bin/python interpreter is able to find the newly created boskel package.

Here are something you may want to catch up:

* **bootstrap.py** Since not everyone has installed zc.buildout locally (by pip install zc.buildout), an initiation script is included. It will download the latest distribute package (it defaults to use setuptools, but distribute package is a replacement and better choice) and zc.buildout packages. It'll generate bin/buildout script, which is the hero of this article.

* **buildout.cfg** A typical buildout configuration file. In detail, the 'develop = .' tells buildout to find pakcage in the current directory, which must includes a setup.py script. And 'interpreter = python' indicates that a new interpreter bin/python should be generated. This special interpreter is made capable of finding all the dependencies of this application.

```bash
$ cat buildout.cfg
[buildout]
parts = python
develop = .

[python]
recipe = zc.recipe.egg
interpreter = python
eggs =
    boskel
```

* **setup.py** A standard package setup script. 'packages' and 'package_dir' option should be written as is shown, since it's a convention of buildout to put all source code in an 'src' folder.

```bash
$ cat setup.py
from setuptools import setup, find_packages

setup(
    name='boskel',
    version='0.1.0',
    packages=find_packages('src'),
    package_dir={'': 'src'},
    zip_safe=False,
    install_requires=[
    ]
)
```

* **offline.cfg** This file will be used to run buildout in a remote server without internet access.

```bash
$ cat offline.cfg
[buildout]
extends = buildout.cfg
download-cache = downloads
install-from-cache = true
```

Production Environment
----------------------

When deploying to a production environment, dependency isolation is very important, unless one plans to deploy only one application per server, then he may install all the packages globally. Otherwise, an environment isolation tool should be chosen, and virtualenv is a good one.

A typical procedure is as below:

```bash
local $ python setup.py sdist
local $ scp xxx-0.1.tar.gz user@remote:/path/to/dist
remote $ source venv/bin/activate
remote $ python setup.py install
```

The dependencies need to be uploaded and installed into the venv first.

As for buildout, we can also use the above approach, deploying with virtualenv. But what if our new distribution's dependency tree is conflict with the previous package. Or I have to setup different virtual environments for production and staging release. So, a better solution is to use buildout for every distribution. 

The main problem of building-out application on remote servers is dependency handling, since most of the time, there won't be internet accessibility in those servers.

There're two solutions, one is setting up a pypi repository with [eggproxy](http://pypi.python.org/pypi/collective.eggproxy). It will cache the eggs downloaded from internet for further use.

Another approach is to install from local cache, as is described below.

zc.buildout has an offline mode (bin/buildout -o, which is also useful when developing, since it will not try to checkout the latest packages, making the building much faster). We can trigger this behavior in configuration file, as is shown in the offline.cfg. The 'download-cache' option indicates where to find the packages. Obviously there should be only one download-cache per server, and whether to use absolute path or a symbolic link to the path is your call.

How about the zc.buildout package itself? In development environment, we use bootstrap.py to install it, but there's not offline-mode bootstrap.py available. So my answer to it is setting up a virtualenv which includes a zc.buildout package. To setup this venv:

```bash
$ cd virtualenv-x.x.x
$ python setup.py install
$ virtualenv venv --distribute
$ source venv/bin/activate
$ cd zc.buildout-x.x.x
$ python setup.py install
```

Later, we can use the venv/bin/buildout script to run buildout (no activation needed).

Here's the deploying procedure:

1. Export the source code from git repository, and rsync it to remote server.
2. Make a symbolic link to the download cache.
3. Execute venv/bin/buildout -c offline.cfg

We can use [Fabric](http://docs.fabfile.org/en/1.4.3/index.html) to do the job. Here's an example task:

```python
from fabric.api import lcd, local, cd, run
from fabric.contrib.project import rsync_project

@task
def deploy(rev='master')
    
    with lcd('~/boskel'):
        local('git clone --mirror git@github.com:jizhang/boskel.git')
        local('git remote update')
        rev = local('git rev-parse %s' % rev)
        prefix = 'boskel-%s' % rev[:7]
        local('git archive --prefix=%s/ --format=tar %s | tar xf - -C %s' %\
              (prefix, rev, '~/boskel/dist'))
    
    rsync_project('/path/to/boskel/dist', '~/boskel/%s' % prefix)
    
    with cd('/path/to/boskel/dist/%s' % prefix):
        run('ln -s -n -f %s downloads' % '/path/to/download-cache')
        run('/path/to/venv/bin/buildout -c offline.cfg')
```

Conclusion
----------

Here we combined buildout and virtualenv to setup isolated environment for every distribution. But two things need to be improved:

1. collective.eggproxy is better than local cache.
2. It's unnecessary to build application in every server. We should use a dedicated server to build the pakcage and rsync these files to other servers.
